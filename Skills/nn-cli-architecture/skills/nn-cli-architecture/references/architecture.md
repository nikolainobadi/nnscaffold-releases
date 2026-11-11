# Architecture Reference

Structural reference for a Swift command-line tool on NnArgumentParser. Rules in `SKILL.md` take precedence when the two disagree. Signatures live in the NnArgumentParser skill; this file says what each type is for and where it lives.

## Contents

1. Entry point
2. Dependencies
3. Command shape
4. Stage payload types and where they live
5. Module boundaries
6. Errors
7. Folder layout
8. Testing shape

---

## 1. Entry point

The root is the only type that conforms to `NnRootCommand`. It holds:

- `static let configuration` listing the subcommands and groups, with a non-empty `abstract` and the tool's `version`.
- `@OptionGroup var interactivity: InteractivityOptions`, declared once here. Validation runs for every command in the parsed chain, so this one declaration registers the mode for every subcommand in either flag position.
- `static var defaultFactory: any ContextFactory` returning `DefaultContextFactory()`.
- `static func makeContext() -> CommandContext`, in an extension, building the context from `contextFactory`. It is also the one place that registers process-wide behavior for the run: an `NnOutputWriter` through `NnOutput.register(writer:)` when a seam (a live progress display) must own the terminal, and `NnInterrupt.register` cleanups. `nnscaffold cli init` registers `ChildProcesses.terminateAll(gracePeriod: 2)` in every tool, and it stays whether or not the tool runs child processes yet. It also registers `ProgressAwareOutputWriter` as the `NnOutput` writer and the progress display's `restoreTerminal()` after the child-process cleanup, so the restore runs first. Any further seam that takes over the terminal registers its restore the same way. Cleanups run in reverse registration order when Ctrl+C or SIGTERM arrives under the root's `main()`.

A **group** (`ship`, `config`) is a plain `ParsableCommand` with only `configuration` and its child subcommands. It declares no options and no `run()`. A group comes from `nnscaffold cli add-group` and a command from `nnscaffold cli add-command`; both register themselves on their parent.

Subcommands are declared as `extension Root { struct Feature: NnWorkflowCommand { … } }`, or `extension Root.Group { … }` for a grouped command, so the type name reads as its command path.

## 2. Dependencies

Three types, all in `Main/`, plus any extra seam the factory vends:

| Type | Role |
| --- | --- |
| `ContextFactory` (protocol) | `makeShell()`, `makeFileSystem()`, `makePicker()`, `makeProgressDisplay()`, `makeAsyncShell()` when a command needs it, and a `make<Seam>()` for each extra seam. Infrastructure only. No feature-specific methods. |
| `DefaultContextFactory` | The production factory. `makePicker()` switches on `InteractionMode.current`: `SwiftPicker(interruptBehavior: .waitForTermination, assumeYes:, permissionSubmitMode: .immediate)` when interactive, the non-interactive picker otherwise, both given the mode's `assumeYes` flag. Passing it to the interactive picker is what makes `--yes` approve confirmations in a terminal; text input and selections still prompt. Waiting for termination is right because the root's `main()` always ends the process on interrupt; the default, `.finishAsCancelled`, would return `nil` from an interrupted prompt and let the workflow carry on. `.immediate` accepts the first `y` or `n` keypress at a confirmation without Enter; piped input still answers with a line, and `--yes` still skips the prompt. An extra seam with a human-facing effect (opening a path in Finder or the browser) switches the same way, vending a skipping variant when non-interactive. Its method returns an outcome type in the seam's file (`PathOpenOutcome`), and both variants return the case for what they did (universal rule 3). `makeProgressDisplay()` returns `LiveProgressDisplay()` only when the mode is interactive and `TerminalCapabilities.standardErrorIsTerminal` is true, and `PlainProgressDisplay()` otherwise. A tool that wants its own colors returns `LiveProgressDisplay(theme:)` with a `ProgressTheme` instead; the plain display takes no theme. |
| `CommandContext` | A struct of the vended primitives (`shell`, `fileSystem`, `picker`, `progressDisplay`, and `asyncShell` when vended). Built once per command by `Root.makeContext()`. Carries no flags and no interactivity value. |

The factory vends an `AsyncShell` beside the `Shell` only when a command runs shell work concurrently or must cancel it. Synchronous work keeps the `Shell`.

Every tool vends `any ProgressDisplay`; `nnscaffold cli init` generates it. `Root.makeContext()` builds it once, registers `ProgressAwareOutputWriter` (in `Main/`), which forwards every `NnOutput` write to the display's `writeOutput(_:)`, and registers its `restoreTerminal()` as an `NnInterrupt` cleanup. `makeWorkflow()` passes it to the use cases that track work, which take `any ProgressDisplay` in `init`. Inspectors, interpreters, collectors, and formatters never receive it. An inspector may live in a library target, which can't depend on CLI code (section 5), so the rule holds wherever the inspector lives.

When an inspector's reads are slow enough that the command would look stalled (a network call, `git fetch`), `makeWorkflow()` wraps it in `NnTrackedInspector`, from NnArgumentParser 1.3.0. The workflow names the wrapper as its inspector type (`let inspector: NnTrackedInspector<FeatureInspector>`), and the inspector itself doesn't change:

```swift
inspector: .init(base: .init(/* inspector args */), track: { work in
    return try await context.progressDisplay.track("Checking the repository") { _ in
        return try await work()
    }
}),
```

The wrapper covers all of `inspect()`, so it shows one message for the whole stage. A fact recorded as missing (a failed fetch stored as `nil` or a `<fact>Failure`) still counts as success and shows `✓`; only a thrown error shows `✗`.

`nnscaffold cli init` generates a factory that vends a shell, a file system, a picker, and a progress display. Adding a seam to it touches five places, all in one change:

1. The `ContextFactory` protocol in the root file gains `make<Seam>()`.
2. `DefaultContextFactory` implements it, switching on interactivity when the seam has a non-interactive variant.
3. `CommandContext` gains a stored property for it.
4. `Root.makeContext()` passes `contextFactory.make<Seam>()` into the context and registers any writer or cleanup the seam needs.
5. The test factory in `Tests/<tool>Tests/Shared/`, when one exists, vends a mock.

A feature that builds the seam itself in `makeWorkflow()` instead bypasses the factory, and a test's factory can no longer replace it.

The primitives come from NnShellKit, NnFileKit, and SwiftPickerKit. A tool may re-export them through its own `<Tool>ShellKit`-style targets so feature code imports one module per concern; that is a packaging choice and changes none of the rules here.

Stores (configuration files, pending-state folders) are not factory products. They are built in `makeWorkflow()` from the file system the factory vended, and injected into the inspector, performer, or use case that needs them.

## 3. Command shape

```swift
extension Root {
    struct Feature: NnWorkflowCommand {
        static let configuration = CommandConfiguration(commandName: "feature", abstract: "…")

        @OptionGroup var options: FeatureOptions

        func makeWorkflow() throws -> FeatureWorkflow {
            let context = Root.makeContext()
            // build stores, inspector, collector, performer from context
            return .init(inspector: …, inspectionInterpreter: .init(input: options.makeInput()), collector: …, collectionInterpreter: .init(), performer: …)
        }

        func handleResults(_ results: FeaturePerformer.Results) throws {
            if let message = FeatureResultsFormatter.message(for: results) {
                NnOutput.write(message)
            }
        }
    }
}
```

- `makeWorkflow()` is where a reader starts: it names every stage the command runs and everything each stage is given.
- What `makeWorkflow()` builds, and what a stage may build itself, is set by universal rules 5 and 6. A dependency two stages share is, for example, one prompter used by both the collector and a use case.
- `<Group>Assembly` (universal rule 7) lives in `<Group>/Shared/Command/<Group>Assembly.swift`. Its static methods return built dependencies, which each `makeWorkflow()` injects. Because it is never part of the factory, a test's factory still replaces every primitive it uses.
- `<Feature>Workflow` is a struct of five `let` properties conforming to `NnWorkflow`. It has no behavior of its own.
- `<Feature>Options` and `<Feature>Input` follow universal rule 9; `validate()` rejects conflicting pairs with `ValidationError`. Every Input field is explicit. Optional fields mean "the caller did not decide this"; the inspection interpreter turns each into a requirement.
- `<Feature>ResultsFormatter` is a caseless enum with `static func message(for:) -> String?`, called from `handleResults` (universal rule 8). Progress or warnings during work are written by the use case that produced them: through the progress display. Progress during inspection is the one exception, and comes from `NnTrackedInspector` (section 2). Text that sets up a prompt (a suggestion to accept, a summary before a confirmation) belongs with that prompt and is written the same way. `NnOutput` routes every write through the registered writer, suppresses writes once an interrupt is being handled, and is the only output `testRun` captures.
- Stage methods may be `async`. `nnscaffold` generates `inspect()` and `perform(brief:)` as synchronous `throws`; a stage that awaits anything changes its signature to `async throws`. Never bridge with `Task { }`: the stage would return before the work finishes and its errors would be lost. The workflow runs on an executor that tolerates blocking, so synchronous shell and file work inside a stage is fine. Where concurrent work lives is set by the Perform stage rule. Command-scoped state is inherited by child tasks but not by `Task.detached` (universal rule 13).

## 4. Stage payload types and where they live

| Payload | Type | Lives in |
| --- | --- | --- |
| Facts | `<Feature>Inspector.Facts` (nested) | with the inspector; in a library target, the inspector's own file |
| Findings | `<Feature>Findings` | `Models/` |
| Requirements | `<Feature>Requirements`, one `NnRequirement<Value, Info>` per missing input | `Models/` |
| Info for an unknown requirement | nested in `<Feature>Requirements` (`Requirements.UnknownVersion`) | `Models/`, same file as `<Feature>Requirements` |
| Answers | `<Feature>Collector.Answers` (nested) | `Collector/<Feature>Collector.swift`, under `// MARK: - Dependencies` |
| Brief | `<Feature>Performer.Brief` (nested) | `Performer/<Feature>Performer.swift`, under `// MARK: - Dependencies` |
| Results and summaries | `<Feature>Performer.Results`, `<Feature>Performer.<Case>Summary` | `Performer/<Feature>Performer.swift`, under `// MARK: - Dependencies` |
| A payload part used by sibling features, held by each one's Findings and Brief | `<Group><Concern>` | `<Group>/Shared/Models/` |
| Flags several commands share | `<Concern>Options: ParsableArguments`, with `makeInput()` | `Shared/Command/` of the closest folder holding every command that uses it |
| Their plain values | `<Concern>Input` | same folder as `<Concern>Options` |
| What the inspection interpreter derives from them | `<Concern>Settings`, held whole by each command's Findings and Brief | `Shared/Models/` of the same folder |

The collection interpreter passes a shared payload part from the findings into the Brief unchanged. The Brief then carries it, so the performer never reaches back into findings.

A `<Concern>Settings` is built in one place: a shared interpreter helper takes the `<Concern>Input` plus any facts it needs, and every inspection interpreter calls it, so the checks behind those settings (such as a missing git author) live once. When the flags pass through unchanged, the Settings fields copy the Input's; keep the type anyway, so the Input still stops at the inspection interpreter.

Requirement and Info types sit in `Models/`, owned by the feature, because both interpreters and the collector read them. Nesting them under the collector would make the inspection side depend on the collector, which breaks the stage rules. Every payload is a `Sendable` value type.

## 5. Module boundaries

Inspectors, stores, detectors, and use cases may live in library targets. A library target doesn't depend on NnArgumentParser, ConsoleProgressKit, or SwiftPickerKit: it holds logic that isn't tied to a terminal, so it stays reusable and testable without one. An inspector there exposes `func inspect() throws -> Facts` (or `async throws` when a read is asynchronous); the executable adopts the stage protocol with an empty extension in `Features/<Feature>/Extensions/`:

```swift
extension ProjectInspector: @retroactive NnInspector { }
```

Interpreters, collectors, prompters, formatters, and the command itself stay in the executable: they are the tool's opinions, not reusable logic. A use case that tracks work also stays in the executable, because it takes `any ProgressDisplay`.

A new library target in the Core package comes from `nnscaffold cli add-kit`, never from hand edits to the Core manifest; `Workflows/add-command.md` step 4a has the invocation.

## 6. Errors

One error enum per feature, `<Feature>Error: Error, Equatable`, with a `LocalizedError` conformance in an extension. Each message tells the user what to do next, not only what went wrong. A failure after the first change also follows **Say where things stand** in the `nn-architecture` skill. It lives in `Features/<Feature>/Workflow/`. A shared helper's error lives in the `Workflow/` of the closest `Shared/` that holds every feature using the helper.

Which stages throw it is set by the stage rules in `SKILL.md`. There is no tool-wide error enum: a feature's failures travel with the feature when it is added or removed.

## 7. Folder layout

```
Sources/<tool>/
  Main/
    <Tool>.swift                    root command and the ContextFactory protocol
    <Seam>.swift, <Variant><Seam>.swift   an extra factory seam's protocol and implementations, one file each; omit when none
    DefaultContextFactory.swift
    CommandContext.swift
    ProgressAwareOutputWriter.swift       the NnOutput writer that forwards to the progress display
  Features/
    <Feature>/
      Command/
        <Feature>.swift             the NnWorkflowCommand
        <Feature>Options.swift      omit when the command has no flags
        <Feature>Input.swift        omit when the command has no flags
      Workflow/
        <Feature>Workflow.swift
        <Feature>Error.swift
      Inspector/
        <Feature>Inspector.swift    the inspector with its nested Facts; omit when it lives in a library target
      Interpreters/
        Inspection/
          <Feature>InspectionInterpreter.swift
          <Field>Detector.swift
        Collection/
          <Feature>CollectionInterpreter.swift
        Shared/                     helpers both interpreters use; omit when empty
      Collector/
        <Feature>Collector.swift
        Prompters/
          <Value>Prompter.swift
      Performer/
        <Feature>Performer.swift
      UseCases/
        <AgentNoun>.swift           a use case, named for its job, such as TagCreator.swift
      FollowUps/
        <Value>Prompter.swift       a follow-up a use case asks once the work has produced something
      Formatter/
        <Feature>ResultsFormatter.swift
      Models/
        <Feature>Findings.swift
        <Feature>Requirements.swift
      Extensions/                   retroactive conformances, small extensions on library types
    <Group>/
      <Group>.swift                 the group command
      <ChildFeature>/               one folder per child, laid out as above
      Shared/                       what two or more children use, by role subfolder
        Command/
          <Group>Assembly.swift     omit when the children share no dependencies
          <Concern>Options.swift    flags the children share; omit when none
          <Concern>Input.swift
        Interpreters/               helpers loose, or split into Inspection/, Collection/, and Shared/
        Models/
          <Concern>Settings.swift
    Shared/                         what features in different groups use, by role subfolder; omit when empty
      Command/
        <Concern>Options.swift      flags commands in different groups share
        <Concern>Input.swift
      Models/
        <Concern>Settings.swift
```

Rules:

- A folder holds files or subfolders, never both. The exceptions are a feature's or group's root file beside its subfolders, and `<Feature>Collector.swift` beside `Prompters/`.
- Omit any role folder the feature does not need. A passthrough command has no `Collector/`, but may have `FollowUps/`.
- A file starts in the feature that needed it and moves to `Shared/` only when a second feature uses it. It goes in the `Shared/` of the closest folder that holds every feature using it: the group's folder when they share a group, `Features/` otherwise.
- A feature is the unit of deletion: removing its folder removes everything it owned.
- Prompters come from `nnscaffold cli add-prompter`. A prompter the collector asks goes into `Collector/Prompters/`, `<Group>/Shared/Prompters/`, or `Shared/Prompters/`. A follow-up prompter a use case asks (`--stage perform`) goes into `FollowUps/`, `<Group>/Shared/FollowUps/`, or `Shared/FollowUps/`; `Performer/` and `UseCases/` hold files, so it gets its own role folder. Interpreter helpers (`<Name>Detector`, `Validator`, `Classifier`, `Builder`) come from `nnscaffold cli add-helper`, into the feature's `Interpreters/Inspection/`, `Interpreters/Collection/`, or `Interpreters/Shared/`, or into `<Group>/Shared/Interpreters/` or `Shared/Interpreters/`. Each also writes its test file in the mirrored test folder.
- `<Group>/Shared/Interpreters/` and `Shared/Interpreters/` are either loose helpers or split into `Inspection/`, `Collection/`, and `Shared/` subfolders, never a mix. `add-helper` follows the shape already there: loose helpers take only `--stage shared`, and a split folder takes any stage. Split a crowded folder by hand; the tool cannot tell a loose helper's stage from where it sits.
- A helper's own result type (what a detector returns) is nested in the helper under `// MARK: - Dependencies` and stays in its file.
- Every leaf command has a flow file at `<tool>/Flows/<command path>.md`, at the package root beside `Sources/`, never inside it. A group is a folder in `Flows/`. Its format is in `references/flow-format.md`, and a feature's removal removes its flow file too.
- Any other value type that is not a stage payload, such as a value an interpreter builds and hands to its helpers, goes in `Models/` of the feature that uses it. When a second feature uses it, it moves to the `Models/` of the closest `Shared/`, like any other file. If Findings or Brief starts holding it, it becomes a payload part under section 4.

## 8. Testing shape

Four kinds of test, in order of preference. Read `TestingReference.md` in the NnArgumentParser skill for the API before writing any of them.

1. **Interpreters as pure functions.** Build the facts (or findings, requirements, answers) by hand, call `interpret`, assert on the result or the thrown `<Feature>Error`. No mocks needed.
2. **Workflow wiring.** Build a `<Feature>Workflow` with `MockInspector`, `MockCollector`, and `MockPerformer` around the real interpreters, run it, and assert on `capturedRequirements` and `capturedBriefs`.
3. **Whole command.** Drive the root with `try await testRun`, passing the test factory vending mock infrastructure, the command's arguments, and the interaction mode explicitly; assert on the captured `NnOutput` output or on what the mocks recorded. Progress written through a display is not captured; the test factory vends `MockProgressDisplay` from the `ConsoleProgressTesting` product, and the test asserts on what it recorded (`trackedTitles`, each activity's `updates` and `outcome`, `notes`). A use-case test injects its own `MockProgressDisplay` the same way. A tracked inspector's title shows up in `trackedTitles` on the factory's display, and the inspector's own tests need no mock display. Interrupt cleanups are tested through `testRunCapturingCleanups`, which returns them without running them; run them and assert `terminalWasRestored` on the factory's display.
4. **Performers and use cases.** Test each use case for its own knowledge, through its public surface, with real collaborators and mocks only for what fronts something outside the program. Test a performer only for its sequence: the order of its steps, what each step receives, and that a failure stops the steps after it. The Testing section of the `nn-architecture` skill has the rule.

Tests live in `Tests/<tool>Tests/`, mirroring the path below `Sources/<tool>/`: `Tests/<tool>Tests/Features/<Feature>/`, or `Tests/<tool>Tests/Features/<Group>/<Feature>/` for a grouped command, with the same role subfolders. The test factory and any shared helper live in `Tests/<tool>Tests/Shared/`. Test code may use default parameter values; production code may not.

`Tests/<tool>Tests/Main/FlowFilesTests.swift` fails when a leaf command has no flow file or a flow file has no command (`references/flow-format.md`). `nnscaffold cli init` writes it, and `nnscaffold cli add-flow-tests` adds it to a tool created before flow files. It is generated and never edited.

`Tests/<tool>Tests/Main/ArchitectureRulesTests.swift` fails when the tool breaks a Don't list item marked Checked, except the flow-file item, and each failure names the file and the rule. Its scans read only the tool's own `Sources/`. `nnscaffold cli init` writes it, and `nnscaffold cli add-architecture-tests` adds it to an existing tool and replaces it when the rules change. It is generated and never edited.

When commands share a `<Concern>Settings`, the test factory has two helpers for it: one that takes every field, for tests that check those fields, and short ones that fix the fields a test does not care about (`make<Concern>Settings(previewRequested:)`). Tests that do not check the shared values use the short helpers, so adding a field to the settings changes only the factory and the tests that check it.

Tests are written when the request asks for them. A workflow step that mentions tests is not permission to add them.
