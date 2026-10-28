---
name: nn-cli-architecture
description: Architecture reference for Swift command-line tools built on NnArgumentParser. Covers the NnRootCommand root with a dependency factory, the NnWorkflow inspect/collect/resolve pipeline that every substantive command follows, where each stage type lives, the feature folder layout, and file style. USE WHEN creating a new Swift CLI, adding or reshaping a command in an existing one, planning or writing an implementation plan for a new command, subcommand, group, or feature of a Swift CLI, deciding where a type or file belongs in a command-line tool, reviewing a Swift CLI's structure, setting up CLI tests, or answering how a command built on NnArgumentParser or NnWorkflow should be organized. Not for the NnArgumentParser API itself (its own skill owns that), SwiftUI apps, or Swift packages that are not command-line tools.
---

# nn-cli-architecture

Rules for how a Swift command-line tool on NnArgumentParser is structured, each paired with the smell that breaks it. The rules name no particular tool; an existing tool that disagrees with a rule is the thing to fix.

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
3. **Interactivity is read in exactly one place.** The default factory picks the real or the non-interactive picker from `InteractionMode.current`. The real picker is built with `interruptBehavior: .waitForTermination`, so Ctrl+C at a prompt ends the tool instead of handing the workflow an empty answer. No stage, use case, or context carries an "is interactive" value. Smell: `isInteractive` anywhere outside `DefaultContextFactory`.
4. **Every substantive command is an `NnWorkflowCommand`.** `makeWorkflow()` is the composition root. From `Root.makeContext()` it builds every store, use case, and client, and every dependency two stages share, and passes each stage the primitives it needs. A stage may build its own private helpers (prompters from the picker, detectors from the file system) when only that stage uses them and they need nothing beyond the primitive it was given. `handleResults` passes the results to `<Feature>ResultsFormatter.message(for:)` and writes the message with `NnOutput.write(_:)`. Nothing else writes the final report.
5. **Options and Input.** Flags live in a `<Feature>Options: ParsableArguments` type that rejects conflicting flags in `validate()` and converts to a plain `<Feature>Input` struct (no ArgumentParser types) through `makeInput()`. The Input reaches the inspection interpreter through its initializer. A command with no flags has neither type.
6. **No default parameter values under `Sources/`.** Every call site spells every argument. Stored-property defaults on `ParsableArguments` are properties, not parameters, and are allowed.
7. **Every command has a non-empty `abstract`.** A command whose prompts have no flag equivalent says so in `discussion`.
8. **Every line the user reads goes through `NnOutput.write(_:)` or the progress display.** The final report, warnings, and text that sets up a prompt go through `NnOutput.write(_:)`. Progress on tracked work (`track`, `note`, activity updates) goes through the `ProgressDisplay` the factory vends; it draws on stderr itself, `testRun` does not capture it, and tests assert on what a mock display recorded. How `NnOutput` is displayed belongs to the writer registered in `Root.makeContext()`: `ProgressAwareOutputWriter` forwards to the display's `writeOutput(_:)` so results never draw over running activities. Smell: `print` or a direct standard-error write under `Sources/`; a `LiveProgressDisplay` or `PlainProgressDisplay` constructed outside `DefaultContextFactory`.
9. **Command-scoped state stays in the command's task tree.** `contextFactory`, `Root.makeContext()`, and `InteractionMode.current` are read from `makeWorkflow()` and the code it calls, never from `Task.detached`, where NnArgumentParser traps.

## Stage rules

The pipeline is inspect → interpret → collect → interpret → resolve. Decisions belong to the two interpreters and nowhere else.

**Inspect** (`<Feature>Inspector`). Gathers facts: tools present, files present, repository state. A missing thing is `nil`, not an error. Throws only when an operation itself fails. Changes nothing the user owns: no writing files, no moving branches, no pushing, no prompting. Reads are allowed, local or over the network, and so is refreshing remote-tracking refs and tags (`git fetch`), since that is a cache refresh. An unreachable source is a fact (`nil` or `false`), not a throw; the inspection interpreter decides whether it blocks. `inspect()` may be `async` when a read is. Smell: an inspector that chooses between options, or one that changes what the user owns to learn something.

**Inspection interpreter** (`<Feature>InspectionInterpreter`). Makes every decision that can be made before the user is asked anything. Throws a `<Feature>Error` for every fact that blocks the command; a blocking fact the inspector collects and the interpreter never checks is a bug. Returns findings (settled) and requirements (missing). Each requirement is an `NnRequirement<Value, Info>`: `.known` when input or facts settle it, `.unknown(Info)` otherwise, where `Info` is a plain struct of the facts the next stage needs (candidates, the latest version). The inspection side knows nothing about the collector: no prompt text, no picker types, no types nested under the collector. A helper that works out one input is `<Field>Detector` with `detect(...)`, created with `nnscaffold cli add-helper`; "select" is reserved for something a person picks. A type that classifies or throws holds findings and is never named `*Facts`. Smell: a requirement type nested in the collector; a `*Selector` nobody prompts for; an interpreter receiving `isInteractive`.

**Collect** (`<Feature>CommandCollector`). Asks only for `.unknown` requirements through `NnRequirement.ask`. One `<Value>Prompter` per question, created with `nnscaffold cli add-prompter`, owns that question's immediate follow-ups (pick "custom", then type the value). The collector sequences prompters and threads earlier answers into later ones; that is its only control flow. Returning early when an earlier answer makes the remaining questions moot (keep the config, so nothing else is asked) counts as sequencing. `Answers` holds only what was asked; a field is `nil` when its requirement was known. Every prompt in a command that has Options passes `flagHint:` naming the flag that supplies the value. The collector never reads facts and never reads interactivity. A command with nothing to ask uses `NnPassthroughCollector` and `NnPassthroughCollectionInterpreter`. Smell: a collector reading findings; a prompt without `flagHint` in a command with Options; an empty hand-written collector.

**Collection interpreter** (`<Feature>CollectionInterpreter`). Resolves each requirement with its answer through `resolve(answer:)`, validates typed answers (throwing `<Feature>Error` on bad input), and builds the `Context` with everything the resolver needs. Smell: a resolver reaching back into findings or requirements.

**Resolve** (`<Feature>CommandResolver`). Does the work through use cases injected in `init`; use-case properties are agent nouns (`releasePublisher`, not `publishRelease`). May prompt only for follow-ups that arise from work already done, such as confirming a suggestion the work produced or a final confirmation. Never prompts for an input the command needed before it started. Such a prompt runs inside the progress display's `suspended { }`. `resolve(context:)` is `async` when a use case it awaits is; concurrent work (a task group) lives inside that use case, not in the resolver. Returns `Results`, usually an enum, and writes nothing. Smell: a use case asking for something a flag could have supplied.

## Routing

| Observable condition | Read, completely, before… |
| --- | --- |
| No `Package.swift` with an executable target depending on NnArgumentParser, or the request is to start a tool | [Workflows/new-cli-project.md](Workflows/new-cli-project.md) before creating any file |
| A root command exists and the request adds or reshapes a command, or asks for a plan to do so | [Workflows/add-command.md](Workflows/add-command.md) before planning or creating any file, including a plan document |
| A file is about to be created or moved | the **Folder layout** and **Stage payload types** sections of [references/architecture.md](references/architecture.md) |
| A Swift file is about to be written | [references/style.md](references/style.md) |
| A test is about to be written | the **Testing shape** section of [references/architecture.md](references/architecture.md) |
| The request is to review, audit, or validate an existing tool, feature, or file against this architecture | [Workflows/review-cli.md](Workflows/review-cli.md) before reading the code under review |
| A "where does this go" or "is this allowed" question with no change requested | answer from this file and [references/architecture.md](references/architecture.md); cite the rule and the smell |

If a routed file cannot be read, say which one and what it governs, and stop the step that depended on it. Continue the steps that do not.

## Don't list

The review checklist, used by [Workflows/review-cli.md](Workflows/review-cli.md) and by the add-command check. Each item is the smell of a rule above or a section of `references/architecture.md`. Report only the items that fail, by number.

**Root and dependencies**

1. More than one `NnRootCommand`, or `InteractivityOptions` declared anywhere but the root.
2. A subcommand not declared as an extension of the root or of its group.
3. A group command with options or a `run()`.
4. A factory method for a feature, or a store vended by the factory.
5. Interactivity read anywhere but the default factory, including an `isInteractive` value passed to a stage, use case, or context.

**Command**

6. A substantive command that is not an `NnWorkflowCommand`.
7. A store, use case, or client, or a dependency two stages share, built outside `makeWorkflow()`.
8. A final report written anywhere but `handleResults`, or composed anywhere but `<Feature>ResultsFormatter`; any `print` or direct standard-error write under `Sources/`.
9. A `<Feature>Workflow` with behavior beyond its five `let`s.
10. Conflicting flags not rejected in `validate()`, an `Input` holding ArgumentParser types, or Options and Input on a command with no flags.
11. An empty `abstract`, or prompts with no flag equivalent and no mention of it in `discussion`.

**Inspect**

12. An inspector that changes what the user owns (writing files, moving branches, pushing, prompting), or one that throws for an unreachable source instead of reporting it as a fact.
13. An inspector that chooses between options, throws for a missing thing instead of returning `nil`, or throws a `<Feature>Error`.

**Inspection interpreter**

14. A blocking fact collected and never thrown on.
15. Prompt text, picker types, or collector-nested types on the inspection side, including a requirement or `Info` type nested in the collector.
16. A classifying or throwing type named `*Facts`.
17. A `*Selector` or "select" name for a choice nobody picks; a computed choice is a `<Field>Detector`.

**Collect**

18. A collector or prompter that reads facts or findings.
19. Control flow in the collector beyond sequencing prompters, threading answers, and returning early when an answer makes the rest moot; follow-ups belong to the prompter.
20. An `Answers` field for something that was not asked.
21. A prompt without `flagHint` in a command that has Options.
22. A hand-written empty collector instead of `NnPassthroughCollector`.

**Collection interpreter and resolve**

23. A `Context` missing something the resolver then looks up from findings or requirements.
24. A resolver or use case prompting for an input the command needed up front.
25. A resolver that writes output, or a use-case property that is not an agent noun.

**Errors, layout, and style**

26. A tool-wide error enum, or an error message that does not say what to do next.
27. A payload type outside the place `references/architecture.md` section 4 gives it.
28. A folder holding both loose files and subfolders (apart from a feature's or group's root file), or a file in `Shared/` that only one feature uses.
29. An interpreter, collector, prompter, or formatter in a library target.
30. Default parameter values under `Sources/`.
31. `isX` Bool names.

**Interruption, concurrency, and progress**

32. An interactive `SwiftPicker` built without `interruptBehavior: .waitForTermination`.
33. An `NnOutput` writer or `NnInterrupt` cleanup registered anywhere but `Root.makeContext()`.
34. The `ChildProcesses.terminateAll(gracePeriod:)` cleanup missing from `Root.makeContext()`.
35. `contextFactory`, `Root.makeContext()`, or `InteractionMode.current` read from `Task.detached`.
36. A task group or other concurrent work in a resolver body instead of a use case.
37. An unstructured `Task { }` in a stage or use case used to reach async work from a synchronous body; the signature becomes `async throws` instead.
38. A `LiveProgressDisplay` or `PlainProgressDisplay` constructed anywhere but `DefaultContextFactory`, or a display whose `restoreTerminal()` is not registered as an `NnInterrupt` cleanup in `Root.makeContext()`.

## Completion

A new project is done when the package builds, `Main/` holds the root, factory, context, and `ProgressAwareOutputWriter`, and the first command follows the add-command workflow. An added command is done when every file in its file table exists in the feature folder, every stage type conforms, `makeWorkflow()` builds every store, use case, client, and shared dependency, the subcommand is registered on its parent, the package builds, and the Don't list passes. Write tests only when the request includes them. Report the files created or changed.
