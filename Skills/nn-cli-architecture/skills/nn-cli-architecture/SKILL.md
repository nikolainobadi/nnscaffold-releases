---
name: nn-cli-architecture
description: Architecture reference for Swift command-line tools built on NnArgumentParser. Covers the NnRootCommand root with a dependency factory, the NnWorkflow inspect/collect/perform pipeline that every command follows, where each stage type lives, the feature folder layout, and file style. USE WHEN creating a new Swift CLI, adding or reshaping a command in an existing one, planning or writing an implementation plan for a new command, subcommand, group, or feature of a Swift CLI, deciding where a type or file belongs in a command-line tool, reviewing a Swift CLI's structure, catching up an existing CLI after changes to this architecture or to nnscaffold (what needs refactoring now), writing or reviewing a command's flow file (Flows/*.md) or adding flow files to an existing CLI, setting up CLI tests, or answering how a command built on NnArgumentParser or NnWorkflow should be organized. Not for the NnArgumentParser API itself (its own skill owns that), SwiftUI apps, or Swift packages that are not command-line tools.
---

# nn-cli-architecture

Rules for how a Swift command-line tool on NnArgumentParser is structured. Principles that hold for any project live in the `nn-architecture` skill. Each rule specific to command-line tools is written once, here; the smells that break them are in the Don't list. The rules name no particular tool; an existing tool that disagrees with a rule is the thing to fix, unless the tool records an approved exception ([references/exceptions.md](references/exceptions.md)).

Paths in this file resolve from the directory containing this `SKILL.md`. Paths in the tool being built are always written as project paths.

## Scope and the sibling skill

This skill governs structure: which types exist, what each may do, and where its file lives. It does not restate API surfaces. Load the **NnArgumentParser** skill for signatures, and read these files of it before the action named:

| Before you… | Read in the NnArgumentParser skill |
| --- | --- |
| write any stage type or a `makeWorkflow()` | `WorkflowReference.md` |
| write the root command, the factory, or interactivity flags | `ApiReference.md` |
| write any test | `TestingReference.md` |

If the NnArgumentParser skill is not loaded, install it with `claude plugin install NnArgumentParser@nn-swift-skills`, then continue. Do not reproduce its content from memory.

The **nn-architecture** skill owns how types are named, how work is decomposed, failure and recovery, and how to request an exception. The rules here say how those principles apply to a command-line tool.

Picker, shell, file-system, and progress-display APIs belong to the SwiftPickerKit, NnShellKit, NnFileKit, and ConsoleProgressKit skills. Before writing code that uses a progress display, load the ConsoleProgressKit skill (`claude plugin install ConsoleProgressKit@nn-swift-skills`).

## Universal rules

1. **One root, one factory.** The `@main` type conforms to `NnRootCommand`, declares `@OptionGroup var interactivity: InteractivityOptions` once, and implements `static var defaultFactory`. Subcommands are declared as extensions of the root (or of a group) so they reach the factory through the root type.
2. **The factory is an infrastructure seam.** It vends primitives: a shell, a file system, a picker, and a progress display, which `nnscaffold cli init` generates in every tool; an async shell when a command runs shell work concurrently or must cancel it; and any further side-effect seam that every feature could use and whose real or skipping variant depends on interactivity (a path opener). `Root.makeContext()` bundles them into a `CommandContext`, and is the only place that registers an `NnOutput` writer or `NnInterrupt` cleanups. A factory method per feature is a service locator; features assemble themselves from the primitives instead. A tool that wants its own progress colors passes a `ProgressTheme` to `LiveProgressDisplay(theme:)` in the default factory; nothing else changes.
3. **Interactivity is read in exactly one place.** The default factory picks the real or the non-interactive picker from `InteractionMode.current`. The real picker is built with `interruptBehavior: .waitForTermination`, so Ctrl+C at a prompt ends the tool instead of handing the workflow an empty answer, and with `permissionSubmitMode: .immediate`, so a confirmation takes a single `y` or `n` keypress. No stage, use case, or brief carries an "is interactive" value. A seam with a skipping variant (a path opener) returns an outcome from the call that may skip, named for the seam (`.opened` or `.skipped`); the performer carries it in `Results`, and the formatter reports what actually happened.
4. **Every command is an `NnWorkflowCommand`.** Every command other than a group runs the pipeline, however small. One shape means a command that grows needs no rewrite, and `nnscaffold` generates the ceremony.
5. **`makeWorkflow()` is the composition root.** From `Root.makeContext()` it builds every store, use case, and client, and every dependency two stages share, and passes each stage the primitives it needs. That includes the use cases one use case gives to another. The exception is a collaborator that can't exist until the brief or a request does: a stage or use case builds it from collaborators `makeWorkflow()` injected plus that value, as `nn-architecture`, Dependencies between modules, allows. Such a collaborator is never a store or client.
6. **A stage builds only its private helpers.** A stage may build its own helpers (prompters from the picker, detectors from the file system) when only that stage uses them and they need nothing beyond the primitive it was given. It may also build a collaborator under rule 5's exception.
7. **`<Group>Assembly` builds what sibling commands share.** When sibling commands in a group build the same use cases, stores, or clients, a `<Group>Assembly` caseless enum in `<Group>/Shared/Command/` may build them. Its only inputs are the `CommandContext` and values `makeWorkflow()` already holds, `makeWorkflow()` is its only caller, and it builds no stage and is never part of the factory.
8. **`handleResults` writes the final report.** It passes the results to `<Feature>ResultsFormatter.message(for:)` and writes the message with `NnOutput.write(_:)`. Nothing else writes the final report.
9. **Options and Input.** Flags live in a `<Feature>Options: ParsableArguments` type that rejects conflicting flags in `validate()` and converts to a plain `<Feature>Input` struct (no ArgumentParser types) through `makeInput()`. The Input reaches the inspection interpreter through its initializer. A command with no flags has neither type. Flags that two or more commands take with the same meaning live once, in a `<Concern>Options` group in the closest `Shared/Command/`. Each command includes it with `@OptionGroup`, its Input holds a `<Concern>Input`, and its Findings and Brief hold the derived `<Concern>Settings` whole (section 4 of `references/architecture.md`).
10. **No default parameter values under `Sources/`.** Every call site spells every argument. Stored-property defaults on `ParsableArguments` are properties, not parameters, and are allowed.
11. **Every command has a non-empty `abstract`.** A command whose prompts have no flag equivalent says so in `discussion`.
12. **Every line the user reads goes through `NnOutput.write(_:)` or the progress display.** The final report, warnings, and text that sets up a prompt go through `NnOutput.write(_:)`. Progress on tracked work (`track`, `note`, activity updates) goes through the `ProgressDisplay` the factory vends; it draws on stderr itself, `testRun` does not capture it, and tests assert on what a mock display recorded. How `NnOutput` is displayed belongs to the writer registered in `Root.makeContext()`: `ProgressAwareOutputWriter` forwards to the display's `writeOutput(_:)` so results never draw over running activities.
13. **Command-scoped state stays in the command's task tree.** `contextFactory`, `Root.makeContext()`, and `InteractionMode.current` are read from `makeWorkflow()` and the code it calls, never from `Task.detached`, where NnArgumentParser traps.
14. **A mode is a command.** A flag that changes which inspector runs or which set of questions the command has is a separate command, grouped with its sibling under a group when they belong together. A flag that answers one of the command's questions, or changes one step of the same work, is a value: a promptable flag skips its prompt but leaves the set of questions unchanged. An `NnWorkflowCommand` runs one pipeline, so two jobs are two commands, and each gets its own `--help`.

## Stage rules

The pipeline is inspect → interpret → collect → interpret → perform. The command's decisions belong to the two interpreters and nowhere else. A rule about a model's own data lives on the model (nn-architecture, Noun kinds), and an interpreter, a use case, or the store's atomic update calls it.

**Inspect** (`<Feature>Inspector`). Gathers facts: tools present, files present, repository state. A missing thing is `nil`, not an error. When the reason it is missing tells the user what to fix (git failed, rather than "not a repository"), the inspector may record it beside the fact as `<fact>Failure: String?` instead of folding it into `nil` with `try?`; the inspection interpreter decides whether that reason blocks. Throws only when an operation itself fails. Changes nothing the user owns: no writing files, no moving branches, no pushing, no prompting. Reads are allowed, local or over the network, and so is refreshing remote-tracking refs and tags (`git fetch`), since that is a cache refresh. An unreachable source is a fact (`nil` or `false`), not a throw; the inspection interpreter decides whether it blocks. `inspect()` may be `async` when a read is. An inspector never takes the progress display, because it may live in a library target that can't depend on CLI code. When its reads are slow enough that the command would look stalled, `makeWorkflow()` wraps it in `NnTrackedInspector` (section 2 of `references/architecture.md`).

**Inspection interpreter** (`<Feature>InspectionInterpreter`). Makes every decision that can be made before the user is asked anything. Throws a `<Feature>Error` for every fact that blocks the command; a blocking fact the inspector collects and the interpreter never checks is a bug. Returns findings (settled) and requirements (missing). Each requirement is an `NnRequirement<Value, Info>`: `.known` when input or facts settle it, `.unknown(Info)` otherwise, where `Info` is a plain struct of the facts the next stage needs (candidates, the latest version). The inspection side knows nothing about the collector: no prompt text, no picker types, no types nested under the collector. A helper that works out one input is `<Field>Detector` with `detect(...)`, created with `nnscaffold cli add-helper`; "select" is reserved for something a person picks. A type that classifies or throws holds findings and is never named `*Facts`.

**Collect** (`<Feature>Collector`). Asks only for `.unknown` requirements through `NnRequirement.ask`. One `<Value>Prompter` per question, created with `nnscaffold cli add-prompter`, owns that question's immediate follow-ups (pick "custom", then type the value). The collector sequences prompters and threads earlier answers into later ones; that is its only control flow. Returning early when an earlier answer makes the remaining questions moot (keep the config, so nothing else is asked) counts as sequencing. `Answers` holds only what was asked; a field is `nil` when its requirement was known. Every prompt in a command that has Options passes `flagHint:` naming the flag that supplies the value. The collector never reads facts and never reads interactivity. A command with nothing to ask uses `NnPassthroughCollector` and `NnPassthroughCollectionInterpreter`.

**Collection interpreter** (`<Feature>CollectionInterpreter`). Resolves each requirement with its answer through `resolve(answer:)`, validates typed answers (throwing `<Feature>Error` on bad input), and builds the `Brief` with everything the performer needs.

**Perform** (`<Feature>Performer`). Does the work through use cases, or a store, injected in `init`. A store is injected directly only when the work is one read or write with no logic around it; once there is a sequence of steps, a second dependency, or progress to report, it becomes a use case. The performer is the coordinator. A coordinating use case exists only when two commands share the sequence or the sequence runs concurrently. A use case is an agent noun (`ReleasePublisher`), and the property holding it is the type name with its whole leading acronym lowercased. May prompt only for follow-ups that arise from work already done, such as confirming a suggestion the work produced or a final confirmation. Never prompts for an input the command needed before it started. Such a prompt runs inside the progress display's `suspended { }`. The follow-up is asked through a `<Value>Prompter` from `nnscaffold cli add-prompter --stage perform`, held by the use case that asks it, and lives in `<Feature>/FollowUps/`. `perform(brief:)` is `async` when a use case it awaits is; concurrent work (a task group) lives inside that use case, not in the performer. Returns `Results`, usually an enum, and writes nothing. When a step fails after an earlier step changed something, whichever runs the steps (the performer, or a coordinating use case) catches the error and throws a `<Feature>Error` case that says what failed, what finished, what didn't, and what to run next (nn-architecture, Say where things stand). The case carries the underlying error's message as a `String`, since `<Feature>Error` is `Equatable`. A use case in the executable throws `<Feature>Error` cases for its own failures, and a shared one throws the error in the `Workflow/` of the closest `Shared/` that holds every feature using it, as a shared helper does; only a use case in a library target, where `<Feature>Error` isn't visible, has its own error type.

**Shared state** (state another process may write between inspect and perform) is changed in one step. The interpreter decides what the command does (join the line, release a session). The performer or use case passes that to the store's atomic update, `update(_:)`, which reads the current state, applies the model's rule, and writes it back. Its shape is `func update<Outcome>(_ change: (inout State) throws -> Outcome) throws -> Outcome`. The store takes the lock with NnFileKit's `FileSystem.withExclusiveLock(at:_:)` and does the read, the rule, and the write inside its closure; it never writes its own `flock` code. Its tests assert the lock request through `MockFileSystem.lockedPaths`, since the mock takes no real lock and can't show mutual exclusion. The model's rule returns the outcome, because another process may have changed the state since the interpreters ran (another session took the lock first). The performer reports that outcome: a `Results` case when it is a normal result (queued behind another session), a thrown `<Feature>Error` only when it stops the command. The inspector's snapshot of that state is for preconditions and the report only, never for the new value. One atomic `update(_:)` counts as the one read or write that lets a store be injected directly into a performer.

## Routing

| Observable condition | Read, completely, before… |
| --- | --- |
| No `Package.swift` with an executable target depending on NnArgumentParser, or the request is to start a tool | [Workflows/new-cli-project.md](Workflows/new-cli-project.md) before creating any file |
| A root command exists and the request adds a flag or argument to an existing command | the **Adding a flag to an existing command** section of [Workflows/add-command.md](Workflows/add-command.md) before changing any file |
| A root command exists and the request adds a use case to an existing command | the **Adding a use case to an existing command** section of [Workflows/add-command.md](Workflows/add-command.md) before changing any file |
| A root command exists and the request adds or reshapes a command, or asks for a plan to do so | [Workflows/add-command.md](Workflows/add-command.md) before planning or creating any file, including a plan document |
| A change to this skill's rules or to what `nnscaffold` generates is being made | [references/changelog.md](references/changelog.md), then add its entry under `## [Unreleased]` in the same change |
| A type is about to be named, or work is about to be split across types | the `nn-architecture` skill and [references/glossary.md](references/glossary.md) before naming or splitting |
| A file is about to be created or moved | the **Folder layout** and **Stage payload types** sections of [references/architecture.md](references/architecture.md) |
| A Swift file is about to be written | [references/style.md](references/style.md) |
| A test is about to be written | the **Testing shape** section of [references/architecture.md](references/architecture.md) |
| A flow file is about to be written, updated, or reviewed, or the request adds flow files to an existing tool | [references/flow-format.md](references/flow-format.md) |
| The request is to bring an existing tool up to date with changes to this architecture or to `nnscaffold`, or to list what needs refactoring after them | [Workflows/catch-up.md](Workflows/catch-up.md) before reading the tool's code |
| The request is to review, audit, or validate an existing tool, feature, or file against this architecture, with no mention of recent changes | [Workflows/review-cli.md](Workflows/review-cli.md) before reading the code under review |
| The user approves an exception to a rule for one tool, or asks whether a departure was recorded as deliberate | [references/exceptions.md](references/exceptions.md) before writing or reading the tool's exceptions file |
| A "where does this go" or "is this allowed" question with no change requested | answer from this file and [references/architecture.md](references/architecture.md); cite the rule and the smell |

If a routed file cannot be read, say which one and what it governs, and stop the step that depended on it. Continue the steps that do not.

## Don't list

The review checklist, used by [Workflows/review-cli.md](Workflows/review-cli.md) and by the add-command check. Each item is the smell of a rule above or a section of `references/architecture.md`. Report only the items that fail, by number.

Each item is marked with how it's enforced. **Generated:** `nnscaffold` produces the correct form, so check only code changed after generation. **Checked:** a test fails when it's broken, so review skips it if the project has that test. **Judgment:** only reading the code can tell.

**Root and dependencies** (universal rules 1 to 3)

1. More than one `NnRootCommand`, or `InteractivityOptions` declared anywhere but the root. [Checked]
2. A subcommand not declared as an extension of the root or of its group. [Generated]
3. A group command with options or a `run()`. [Generated]
4. A factory method for a feature, or a store vended by the factory. [Judgment]
5. Interactivity read anywhere but the default factory, including an `isInteractive` value passed to a stage, use case, or brief. [Judgment]
6. A seam with a skipping variant that returns no outcome, or a report announcing an effect the seam skipped. [Judgment]

**Command** (universal rules 4 to 9, 11, 12, and 14)

7. A command other than a group that is not an `NnWorkflowCommand`. [Generated]
8. A store, use case, or client, or a dependency two stages share, built outside `makeWorkflow()` or its group's `<Group>Assembly`, other than a collaborator that is neither a store nor a client, built from injected collaborators plus the brief or a request (universal rule 5). [Judgment]
9. A `<Group>Assembly` that builds a stage, reads interactivity, takes more than the `CommandContext` and values `makeWorkflow()` holds, or is called from anywhere but `makeWorkflow()`. [Judgment]
10. A final report written anywhere but `handleResults`, or composed anywhere but `<Feature>ResultsFormatter` [Judgment]; any `print` under `Sources/` [Checked]; any direct standard-error write under `Sources/` [Judgment].
11. A `<Feature>Workflow` with behavior beyond its five `let`s. [Generated]
12. Conflicting flags not rejected in `validate()`, an `Input` holding ArgumentParser types, or Options and Input on a command with no flags; the same flag with the same meaning declared in two commands' Options instead of a shared `<Concern>Options`, or a shared settings value split back into separate Findings or Brief fields. [Judgment]
13. An empty `abstract` [Generated], or prompts with no flag equivalent and no mention of it in `discussion` [Judgment].
14. A mode kept as a flag instead of a separate command (universal rule 14): an enum or Bool that changes which inspector runs or which questions are asked. Switching on it in more than one stage, or in `makeWorkflow()`, is the usual sign. A value flag that changes one step, such as `--force`, may be read by every stage that step touches. [Judgment]

**Inspect** (the Inspect stage rule)

15. An inspector that changes what the user owns (writing files, moving branches, pushing, prompting), or one that throws for an unreachable source instead of reporting it as a fact. [Judgment]
16. An inspector that chooses between options, throws for a missing thing instead of returning `nil`, or throws a `<Feature>Error`. [Judgment]
17. A `try?` in an inspector that folds a failure into `nil` when its reason is what the user needs to fix it. [Judgment]

**Inspection interpreter** (its stage rule)

18. A blocking fact collected and never thrown on. [Judgment]
19. Prompt text, picker types, or collector-nested types on the inspection side, including a requirement or `Info` type nested in the collector. [Judgment]
20. A classifying or throwing type named `*Facts`. [Judgment]
21. A `*Selector` or "select" name for a choice nobody picks; a computed choice is a `<Field>Detector`. [Judgment]

**Collect** (its stage rule)

22. A collector or prompter that reads facts or findings. [Judgment]
23. Control flow in the collector beyond sequencing prompters, threading answers, and returning early when an answer makes the rest moot; follow-ups belong to the prompter. [Judgment]
24. An `Answers` field for something that was not asked. [Judgment]
25. A prompt without `flagHint` in a command that has Options. [Judgment]
26. A hand-written empty collector instead of `NnPassthroughCollector`. [Generated]

**Collection interpreter and perform** (their stage rules)

27. A `Brief` missing something the performer then looks up from findings or requirements. [Judgment]
28. A performer or use case prompting for an input the command needed up front. [Judgment]
29. A performer that writes output, or a use case that is not named as an agent noun. [Judgment]
30. A use case whose `run` only forwards one call to a store, or a store injected into a performer for work beyond one read or write. [Judgment]

**Errors, layout, and style** (universal rule 10; `references/architecture.md` sections 4, 5, 6, and 7; `references/style.md`)

31. A tool-wide error enum [Generated], or an error message that does not say what to do next [Judgment].
32. A payload type outside the place `references/architecture.md` section 4 gives it. [Generated]
33. A folder holding both loose files and subfolders (apart from a feature's or group's root file, or `<Feature>Collector.swift` beside `Prompters/`), a file in `Shared/` that only one feature uses, or a prompter in the wrong role folder: one a use case asks in `Collector/Prompters/`, or one the collector asks in `FollowUps/`. [Judgment]
34. An interpreter, collector, prompter, or formatter in a library target, or a library target that depends on NnArgumentParser, ConsoleProgressKit, or SwiftPickerKit. [Checked]
35. Default parameter values under `Sources/`. [Judgment]
36. `isX` names on stored Bool properties. [Judgment]

**Interruption, concurrency, and progress** (universal rules 2, 3, 12, and 13; the Inspect and Perform stage rules)

37. An interactive `SwiftPicker` built without `interruptBehavior: .waitForTermination`. [Checked]
38. An `NnOutput` writer or `NnInterrupt` cleanup registered anywhere but `Root.makeContext()`. [Checked]
39. The `ChildProcesses.terminateAll(gracePeriod:)` cleanup missing from `Root.makeContext()`. [Generated]
40. `contextFactory`, `Root.makeContext()`, or `InteractionMode.current` read from `Task.detached`. [Judgment]
41. A task group or other concurrent work in a performer body instead of a use case. [Judgment]
42. An unstructured `Task { }` in a stage or use case used to reach async work from a synchronous body; the signature becomes `async throws` instead. [Checked]
43. A `LiveProgressDisplay` or `PlainProgressDisplay` constructed anywhere but `DefaultContextFactory` [Checked], or a display whose `restoreTerminal()` is not registered as an `NnInterrupt` cleanup in `Root.makeContext()` [Judgment].
44. An inspector that takes a progress display, in any target. [Judgment]

**Flow files** (`references/flow-format.md`)

45. A leaf command without `Flows/<command path>.md`, or a flow file with no command [Checked]; a flow file whose steps, questions, stops, or Code line disagree with the code [Judgment].

**Names** (`nn-architecture`, Choosing a name, rule 6)

46. A type a project named with a catch-all word (Manager, Helper, Handler, Processor, Service, Util, or Utility) [Generated].

**Shared state** (the shared state rule)

47. A read and write of state other processes write, done as separate steps instead of one atomic store update. [Judgment]

**Partial failure** (the Perform stage rule)

48. A step that fails after an earlier step changed something, with its error passed through instead of rethrown as a `<Feature>Error` case naming what failed, what finished, and what to run next. [Judgment]

## Completion

A new project is done when the package builds, `Main/` holds the root, factory, context, and `ProgressAwareOutputWriter`, and the first command follows the add-command workflow. An added command is done when every file in its file table exists in the feature folder, every stage type conforms, `makeWorkflow()` (or its group's `<Group>Assembly`) builds every store, use case, client, and shared dependency, the subcommand is registered on its parent, the package builds, the Don't list passes, and its flow file in `Flows/` describes what was built (add-command step 6). Write tests only when the request includes them. Report the files created or changed.
