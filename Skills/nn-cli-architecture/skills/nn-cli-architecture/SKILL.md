---
name: nn-cli-architecture
description: Architecture reference for Swift command-line tools built on NnArgumentParser. Covers the NnRootCommand root with a dependency factory, the NnWorkflow inspect/collect/resolve pipeline that every command follows, where each stage type lives, the feature folder layout, and file style. USE WHEN creating a new Swift CLI, adding or reshaping a command in an existing one, planning or writing an implementation plan for a new command, subcommand, group, or feature of a Swift CLI, deciding where a type or file belongs in a command-line tool, reviewing a Swift CLI's structure, setting up CLI tests, or answering how a command built on NnArgumentParser or NnWorkflow should be organized. Not for the NnArgumentParser API itself (its own skill owns that), SwiftUI apps, or Swift packages that are not command-line tools.
---

# nn-cli-architecture

Rules for how a Swift command-line tool on NnArgumentParser is structured. Each rule is written once, here; the smells that break them are in the Don't list. The rules name no particular tool; an existing tool that disagrees with a rule is the thing to fix.

Paths in this file resolve from the directory containing this `SKILL.md`. Paths in the tool being built are always written as project paths.

## Scope and the sibling skill

This skill governs structure: which types exist, what each may do, and where its file lives. It does not restate API surfaces. Load the **NnArgumentParser** skill for signatures, and read these files of it before the action named:

| Before you… | Read in the NnArgumentParser skill |
| --- | --- |
| write any stage type or a `makeWorkflow()` | `WorkflowReference.md` |
| write the root command, the factory, or interactivity flags | `ApiReference.md` |
| write any test | `TestingReference.md` |

If the NnArgumentParser skill is not loaded, install it with `claude plugin install NnArgumentParser@nn-swift-skills`, then continue. Do not reproduce its content from memory.

Picker, shell, file-system, and progress-display APIs belong to the SwiftPickerKit, NnShellKit, NnFileKit, and ConsoleProgressKit skills. Before writing code that uses a progress display, load the ConsoleProgressKit skill (`claude plugin install ConsoleProgressKit@nn-swift-skills`).

## Universal rules

1. **One root, one factory.** The `@main` type conforms to `NnRootCommand`, declares `@OptionGroup var interactivity: InteractivityOptions` once, and implements `static var defaultFactory`. Subcommands are declared as extensions of the root (or of a group) so they reach the factory through the root type.
2. **The factory is an infrastructure seam.** It vends primitives: a shell, a file system, a picker, and a progress display, which `nnscaffold cli init` generates in every tool; an async shell when a command runs shell work concurrently or must cancel it; and any further side-effect seam that every feature could use and whose real or skipping variant depends on interactivity (a path opener). `Root.makeContext()` bundles them into a `CommandContext`, and is the only place that registers an `NnOutput` writer or `NnInterrupt` cleanups. A factory method per feature is a service locator; features assemble themselves from the primitives instead.
3. **Interactivity is read in exactly one place.** The default factory picks the real or the non-interactive picker from `InteractionMode.current`. The real picker is built with `interruptBehavior: .waitForTermination`, so Ctrl+C at a prompt ends the tool instead of handing the workflow an empty answer. No stage, use case, or context carries an "is interactive" value. A seam with a skipping variant (a path opener) returns an outcome from the call that may skip, named for the seam (`.opened` or `.skipped`); the resolver carries it in `Results`, and the formatter reports what actually happened.
4. **Every command is an `NnWorkflowCommand`.** Every command other than a group runs the pipeline, however small. One shape means a command that grows needs no rewrite, and `nnscaffold` generates the ceremony.
5. **`makeWorkflow()` is the composition root.** From `Root.makeContext()` it builds every store, use case, and client, and every dependency two stages share, and passes each stage the primitives it needs.
6. **A stage builds only its private helpers.** A stage may build its own helpers (prompters from the picker, detectors from the file system) when only that stage uses them and they need nothing beyond the primitive it was given.
7. **`<Group>Assembly` builds what sibling commands share.** When sibling commands in a group build the same use cases, stores, or clients, a `<Group>Assembly` caseless enum in `<Group>/Shared/Command/` may build them. Its only inputs are the `CommandContext` and values `makeWorkflow()` already holds, `makeWorkflow()` is its only caller, and it builds no stage and is never part of the factory.
8. **`handleResults` writes the final report.** It passes the results to `<Feature>ResultsFormatter.message(for:)` and writes the message with `NnOutput.write(_:)`. Nothing else writes the final report.
9. **Options and Input.** Flags live in a `<Feature>Options: ParsableArguments` type that rejects conflicting flags in `validate()` and converts to a plain `<Feature>Input` struct (no ArgumentParser types) through `makeInput()`. The Input reaches the inspection interpreter through its initializer. A command with no flags has neither type.
10. **No default parameter values under `Sources/`.** Every call site spells every argument. Stored-property defaults on `ParsableArguments` are properties, not parameters, and are allowed.
11. **Every command has a non-empty `abstract`.** A command whose prompts have no flag equivalent says so in `discussion`.
12. **Every line the user reads goes through `NnOutput.write(_:)` or the progress display.** The final report, warnings, and text that sets up a prompt go through `NnOutput.write(_:)`. Progress on tracked work (`track`, `note`, activity updates) goes through the `ProgressDisplay` the factory vends; it draws on stderr itself, `testRun` does not capture it, and tests assert on what a mock display recorded. How `NnOutput` is displayed belongs to the writer registered in `Root.makeContext()`: `ProgressAwareOutputWriter` forwards to the display's `writeOutput(_:)` so results never draw over running activities.
13. **Command-scoped state stays in the command's task tree.** `contextFactory`, `Root.makeContext()`, and `InteractionMode.current` are read from `makeWorkflow()` and the code it calls, never from `Task.detached`, where NnArgumentParser traps.
14. **A mode is a command.** A flag that changes which inspector runs or which set of questions the command has is a separate command, grouped with its sibling under a group when they belong together. A flag that answers one of the command's questions, or changes one step of the same work, is a value: a promptable flag skips its prompt but leaves the set of questions unchanged. An `NnWorkflowCommand` runs one pipeline, so two jobs are two commands, and each gets its own `--help`.

## Stage rules

The pipeline is inspect → interpret → collect → interpret → resolve. Decisions belong to the two interpreters and nowhere else.

**Inspect** (`<Feature>Inspector`). Gathers facts: tools present, files present, repository state. A missing thing is `nil`, not an error. When the reason it is missing tells the user what to fix (git failed, rather than "not a repository"), the inspector may record it beside the fact as `<fact>Failure: String?` instead of folding it into `nil` with `try?`; the inspection interpreter decides whether that reason blocks. Throws only when an operation itself fails. Changes nothing the user owns: no writing files, no moving branches, no pushing, no prompting. Reads are allowed, local or over the network, and so is refreshing remote-tracking refs and tags (`git fetch`), since that is a cache refresh. An unreachable source is a fact (`nil` or `false`), not a throw; the inspection interpreter decides whether it blocks. `inspect()` may be `async` when a read is.

**Inspection interpreter** (`<Feature>InspectionInterpreter`). Makes every decision that can be made before the user is asked anything. Throws a `<Feature>Error` for every fact that blocks the command; a blocking fact the inspector collects and the interpreter never checks is a bug. Returns findings (settled) and requirements (missing). Each requirement is an `NnRequirement<Value, Info>`: `.known` when input or facts settle it, `.unknown(Info)` otherwise, where `Info` is a plain struct of the facts the next stage needs (candidates, the latest version). The inspection side knows nothing about the collector: no prompt text, no picker types, no types nested under the collector. A helper that works out one input is `<Field>Detector` with `detect(...)`, created with `nnscaffold cli add-helper`; "select" is reserved for something a person picks. A type that classifies or throws holds findings and is never named `*Facts`.

**Collect** (`<Feature>CommandCollector`). Asks only for `.unknown` requirements through `NnRequirement.ask`. One `<Value>Prompter` per question, created with `nnscaffold cli add-prompter`, owns that question's immediate follow-ups (pick "custom", then type the value). The collector sequences prompters and threads earlier answers into later ones; that is its only control flow. Returning early when an earlier answer makes the remaining questions moot (keep the config, so nothing else is asked) counts as sequencing. `Answers` holds only what was asked; a field is `nil` when its requirement was known. Every prompt in a command that has Options passes `flagHint:` naming the flag that supplies the value. The collector never reads facts and never reads interactivity. A command with nothing to ask uses `NnPassthroughCollector` and `NnPassthroughCollectionInterpreter`.

**Collection interpreter** (`<Feature>CollectionInterpreter`). Resolves each requirement with its answer through `resolve(answer:)`, validates typed answers (throwing `<Feature>Error` on bad input), and builds the `Context` with everything the resolver needs.

**Resolve** (`<Feature>CommandResolver`). Does the work through use cases, or a store, injected in `init`. A store is injected directly only when the work is one read or write with no logic around it; once there is a sequence of steps, a second dependency, or progress to report, it becomes a use case. Use-case properties are agent nouns (`releasePublisher`, not `publishRelease`). May prompt only for follow-ups that arise from work already done, such as confirming a suggestion the work produced or a final confirmation. Never prompts for an input the command needed before it started. Such a prompt runs inside the progress display's `suspended { }`. `resolve(context:)` is `async` when a use case it awaits is; concurrent work (a task group) lives inside that use case, not in the resolver. Returns `Results`, usually an enum, and writes nothing.

## Routing

| Observable condition | Read, completely, before… |
| --- | --- |
| No `Package.swift` with an executable target depending on NnArgumentParser, or the request is to start a tool | [Workflows/new-cli-project.md](Workflows/new-cli-project.md) before creating any file |
| A root command exists and the request adds a flag or argument to an existing command | the **Adding a flag to an existing command** section of [Workflows/add-command.md](Workflows/add-command.md) before changing any file |
| A root command exists and the request adds or reshapes a command, or asks for a plan to do so | [Workflows/add-command.md](Workflows/add-command.md) before planning or creating any file, including a plan document |
| A file is about to be created or moved | the **Folder layout** and **Stage payload types** sections of [references/architecture.md](references/architecture.md) |
| A Swift file is about to be written | [references/style.md](references/style.md) |
| A test is about to be written | the **Testing shape** section of [references/architecture.md](references/architecture.md) |
| The request is to review, audit, or validate an existing tool, feature, or file against this architecture | [Workflows/review-cli.md](Workflows/review-cli.md) before reading the code under review |
| A "where does this go" or "is this allowed" question with no change requested | answer from this file and [references/architecture.md](references/architecture.md); cite the rule and the smell |

If a routed file cannot be read, say which one and what it governs, and stop the step that depended on it. Continue the steps that do not.

## Don't list

The review checklist, used by [Workflows/review-cli.md](Workflows/review-cli.md) and by the add-command check. Each item is the smell of a rule above or a section of `references/architecture.md`. Report only the items that fail, by number.

**Root and dependencies** (universal rules 1 to 3)

1. More than one `NnRootCommand`, or `InteractivityOptions` declared anywhere but the root.
2. A subcommand not declared as an extension of the root or of its group.
3. A group command with options or a `run()`.
4. A factory method for a feature, or a store vended by the factory.
5. Interactivity read anywhere but the default factory, including an `isInteractive` value passed to a stage, use case, or context.
6. A seam with a skipping variant that returns no outcome, or a report announcing an effect the seam skipped.

**Command** (universal rules 4 to 9, 11, 12, and 14)

7. A command other than a group that is not an `NnWorkflowCommand`.
8. A store, use case, or client, or a dependency two stages share, built outside `makeWorkflow()` or its group's `<Group>Assembly`.
9. A `<Group>Assembly` that builds a stage, reads interactivity, takes more than the `CommandContext` and values `makeWorkflow()` holds, or is called from anywhere but `makeWorkflow()`.
10. A final report written anywhere but `handleResults`, or composed anywhere but `<Feature>ResultsFormatter`; any `print` or direct standard-error write under `Sources/`.
11. A `<Feature>Workflow` with behavior beyond its five `let`s.
12. Conflicting flags not rejected in `validate()`, an `Input` holding ArgumentParser types, or Options and Input on a command with no flags.
13. An empty `abstract`, or prompts with no flag equivalent and no mention of it in `discussion`.
14. A mode enum or Bool switched on in more than one stage, or checked again in `makeWorkflow()`.

**Inspect** (the Inspect stage rule)

15. An inspector that changes what the user owns (writing files, moving branches, pushing, prompting), or one that throws for an unreachable source instead of reporting it as a fact.
16. An inspector that chooses between options, throws for a missing thing instead of returning `nil`, or throws a `<Feature>Error`.
17. A `try?` in an inspector that folds a failure into `nil` when its reason is what the user needs to fix it.

**Inspection interpreter** (its stage rule)

18. A blocking fact collected and never thrown on.
19. Prompt text, picker types, or collector-nested types on the inspection side, including a requirement or `Info` type nested in the collector.
20. A classifying or throwing type named `*Facts`.
21. A `*Selector` or "select" name for a choice nobody picks; a computed choice is a `<Field>Detector`.

**Collect** (its stage rule)

22. A collector or prompter that reads facts or findings.
23. Control flow in the collector beyond sequencing prompters, threading answers, and returning early when an answer makes the rest moot; follow-ups belong to the prompter.
24. An `Answers` field for something that was not asked.
25. A prompt without `flagHint` in a command that has Options.
26. A hand-written empty collector instead of `NnPassthroughCollector`.

**Collection interpreter and resolve** (their stage rules)

27. A `Context` missing something the resolver then looks up from findings or requirements.
28. A resolver or use case prompting for an input the command needed up front.
29. A resolver that writes output, or a use-case property that is not an agent noun.
30. A use case whose `run` only forwards one call to a store, or a store injected into a resolver for work beyond one read or write.

**Errors, layout, and style** (universal rule 10; `references/architecture.md` sections 4, 5, 6, and 7; `references/style.md`)

31. A tool-wide error enum, or an error message that does not say what to do next.
32. A payload type outside the place `references/architecture.md` section 4 gives it.
33. A folder holding both loose files and subfolders (apart from a feature's or group's root file), or a file in `Shared/` that only one feature uses.
34. An interpreter, collector, prompter, or formatter in a library target.
35. Default parameter values under `Sources/`.
36. `isX` Bool names.

**Interruption, concurrency, and progress** (universal rules 2, 3, 12, and 13; the Resolve stage rule)

37. An interactive `SwiftPicker` built without `interruptBehavior: .waitForTermination`.
38. An `NnOutput` writer or `NnInterrupt` cleanup registered anywhere but `Root.makeContext()`.
39. The `ChildProcesses.terminateAll(gracePeriod:)` cleanup missing from `Root.makeContext()`.
40. `contextFactory`, `Root.makeContext()`, or `InteractionMode.current` read from `Task.detached`.
41. A task group or other concurrent work in a resolver body instead of a use case.
42. An unstructured `Task { }` in a stage or use case used to reach async work from a synchronous body; the signature becomes `async throws` instead.
43. A `LiveProgressDisplay` or `PlainProgressDisplay` constructed anywhere but `DefaultContextFactory`, or a display whose `restoreTerminal()` is not registered as an `NnInterrupt` cleanup in `Root.makeContext()`.

## Completion

A new project is done when the package builds, `Main/` holds the root, factory, context, and `ProgressAwareOutputWriter`, and the first command follows the add-command workflow. An added command is done when every file in its file table exists in the feature folder, every stage type conforms, `makeWorkflow()` (or its group's `<Group>Assembly`) builds every store, use case, client, and shared dependency, the subcommand is registered on its parent, the package builds, and the Don't list passes. Write tests only when the request includes them. Report the files created or changed.
