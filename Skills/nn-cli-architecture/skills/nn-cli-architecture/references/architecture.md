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

A **group** (`ship`, `config`) is a plain `ParsableCommand` with only `configuration` and its child subcommands. It declares no options and no `run()`.

Subcommands are declared as `extension Root { struct Feature: NnWorkflowCommand { … } }`, or `extension Root.Group { … }` for a grouped command, so the type name reads as its command path.

## 2. Dependencies

Three types, all in `Main/`, plus any extra seam the factory vends:

| Type | Role |
| --- | --- |
| `ContextFactory` (protocol) | `makeShell()`, `makeFileSystem()`, `makePicker()`, `makeProgressDisplay()`, `makeAsyncShell()` when a command needs it, and a `make<Seam>()` for each extra seam. Infrastructure only. No feature-specific methods. |
| `DefaultContextFactory` | The production factory. `makePicker()` switches on `InteractionMode.current`: `SwiftPicker(interruptBehavior: .waitForTermination)` when interactive, the non-interactive picker with the `assumeYes` flag otherwise. Waiting for termination is right because the root's `main()` always ends the process on interrupt; the default, `.finishAsCancelled`, would return `nil` from an interrupted prompt and let the workflow carry on. An extra seam with a human-facing effect (opening a path in Finder or the browser) switches the same way, vending a skipping variant when non-interactive. `makeProgressDisplay()` returns `LiveProgressDisplay()` only when the mode is interactive and `TerminalCapabilities.standardErrorIsTerminal` is true, and `PlainProgressDisplay()` otherwise. This is the only place interactivity is read. |
| `CommandContext` | A struct of the vended primitives (`shell`, `fileSystem`, `picker`, `progressDisplay`, and `asyncShell` when vended). Built once per command by `Root.makeContext()`. Carries no flags and no interactivity value. |

The factory vends an `AsyncShell` beside the `Shell` only when a command runs shell work concurrently or must cancel it. Synchronous work keeps the `Shell`.

Every tool vends `any ProgressDisplay`; `nnscaffold cli init` generates it. `Root.makeContext()` builds it once, registers `ProgressAwareOutputWriter` (in `Main/`), which forwards every `NnOutput` write to the display's `writeOutput(_:)`, and registers its `restoreTerminal()` as an `NnInterrupt` cleanup. `makeWorkflow()` passes it to the use cases that track work, which take `any ProgressDisplay` in `init`. Inspectors, interpreters, collectors, and formatters never receive it.

`nnscaffold cli init` generates a factory that vends a shell, a file system, a picker, and a progress display. Adding a seam to it touches five places, all in one change:

1. The `ContextFactory` protocol in the root file gains `make<Seam>()`.
2. `DefaultContextFactory` implements it, switching on interactivity when the seam has a non-interactive variant.
3. `CommandContext` gains a stored property for it.
4. `Root.makeContext()` passes `contextFactory.make<Seam>()` into the context and registers any writer or cleanup the seam needs.
5. The test factory in `Tests/<tool>Tests/Shared/`, when one exists, vends a mock.

A feature that builds the seam itself in `makeWorkflow()` instead bypasses the factory, and a test's factory can no longer replace it.

The primitives come from NnShellKit, NnFileKit, and SwiftPickerKit. A tool may re-export them through its own `<Tool>ShellKit`-style targets so feature code imports one module per concern; that is a packaging choice and changes none of the rules here.

Stores (configuration files, pending-state folders) are not factory products. They are built in `makeWorkflow()` from the file system the factory vended, and injected into the inspector, resolver, or use case that needs them.

## 3. Command shape

```swift
extension Root {
    struct Feature: NnWorkflowCommand {
        static let configuration = CommandConfiguration(commandName: "feature", abstract: "…")

        @OptionGroup var options: FeatureOptions

        func makeWorkflow() throws -> FeatureWorkflow {
            let context = Root.makeContext()
            // build stores, inspector, collector, resolver from context
            return .init(inspector: …, inspectionInterpreter: .init(input: options.makeInput()), collector: …, collectionInterpreter: .init(), resolver: …)
        }

        func handleResults(_ results: FeatureCommandResolver.Results) throws {
            if let message = FeatureResultsFormatter.message(for: results) {
                NnOutput.write(message)
            }
        }
    }
}
```

- `makeWorkflow()` is the composition root. Every store, use case, and client is built here from the context, and so is any dependency two stages share (one prompter used by both the collector and a use case); each is injected into the stages that use it. A stage may build its own private helpers from a primitive it was given, such as a collector building its prompters from the picker, when nothing else uses them and they need nothing more.
- `<Feature>Workflow` is a struct of five `let` properties conforming to `NnWorkflow`. It has no behavior of its own.
- `<Feature>Options` groups flags, rejects conflicting pairs in `validate()` with `ValidationError`, and converts to `<Feature>Input` in `makeInput()`. `<Feature>Input` is a plain struct with every field explicit and no ArgumentParser types. Optional fields mean "the caller did not decide this"; the inspection interpreter turns each into a requirement.
- `<Feature>ResultsFormatter` is a caseless enum with `static func message(for:) -> String?`. It is the only place the final report is composed. Progress or warnings during work are written by the use case that produced them: through the progress display. Text that sets up a prompt (a suggestion to accept, a summary before a confirmation) belongs with that prompt and is written the same way. `NnOutput` routes every write through the registered writer, suppresses writes once an interrupt is being handled, and is the only output `testRun` captures.
- Stage methods may be `async`. `nnscaffold` generates `inspect()` and `resolve(context:)` as synchronous `throws`; a stage that awaits anything changes its signature to `async throws`. Never bridge with `Task { }`: the stage would return before the work finishes and its errors would be lost. The workflow runs on an executor that tolerates blocking, so synchronous shell and file work inside a stage is fine. Concurrent work (a task group) lives inside a use case, which the resolver awaits. Command-scoped state (`contextFactory`, `InteractionMode.current`) is inherited by child tasks but not by `Task.detached`, and reading it there traps.

## 4. Stage payload types and where they live

| Payload | Type | Lives in |
| --- | --- | --- |
| Facts | `<Feature>Inspector.Facts` (nested) | with the inspector; in a library target, the inspector's own file |
| Findings | `<Feature>Findings` | `Models/` |
| Requirements | `<Feature>Requirements`, one `NnRequirement<Value, Info>` per missing input | `Models/` |
| Info for an unknown requirement | nested in `<Feature>Requirements` (`Requirements.UnknownVersion`) | `Models/`, same file as `<Feature>Requirements` |
| Answers | `<Feature>CommandCollector.Answers` (nested) | `Collector/<Feature>CommandCollector.swift`, under `// MARK: - Dependencies` |
| Context | `<Feature>CommandResolver.Context` (nested) | `Resolver/<Feature>CommandResolver.swift`, under `// MARK: - Dependencies` |
| Results and summaries | `<Feature>CommandResolver.Results`, `<Feature>CommandResolver.<Case>Summary` | `Resolver/<Feature>CommandResolver.swift`, under `// MARK: - Dependencies` |

Requirement and Info types sit in `Models/`, owned by the feature, because both interpreters and the collector read them. Nesting them under the collector would make the inspection side depend on the collector, which breaks the stage rules. Every payload is a `Sendable` value type.

## 5. Module boundaries

Inspectors, stores, detectors, and use cases may live in library targets that do not depend on NnArgumentParser. An inspector there exposes `func inspect() throws -> Facts` (or `async throws` when a read is asynchronous); the executable adopts the stage protocol with an empty extension in `Features/<Feature>/Extensions/`:

```swift
extension ProjectInspector: @retroactive NnCommandInspector { }
```

Interpreters, collectors, prompters, formatters, and the command itself stay in the executable: they are the tool's opinions, not reusable logic.

A new library target in the Core package comes from `nnscaffold add-kit`, never from hand edits to the Core manifest; `Workflows/add-command.md` step 4a has the invocation.

## 6. Errors

One error enum per feature, `<Feature>Error: Error, Equatable`, with a `LocalizedError` conformance in an extension. Each message tells the user what to do next, not only what went wrong. It lives in `Features/<Feature>/Workflow/`.

The inspection interpreter throws it for blocking facts; the collection interpreter throws it for invalid answers. The inspector never throws it. There is no tool-wide error enum: a feature's failures travel with the feature when it is added or removed.

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
        <Feature>CommandCollector.swift
        Prompters/
          <Value>Prompter.swift
      Resolver/
        <Feature>CommandResolver.swift
      UseCases/
        <Verb><Noun>UseCase.swift
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
    Shared/                         what features in different groups use, by role subfolder; omit when empty
```

Rules:

- A folder holds files or subfolders, never both. The one exception is a feature's or group's root file beside its subfolders.
- Omit any role folder the feature does not need. A passthrough command has no `Collector/`.
- A file starts in the feature that needed it and moves to `Shared/` only when a second feature uses it. It goes in the `Shared/` of the closest folder that holds every feature using it: the group's folder when they share a group, `Features/` otherwise.
- A feature is the unit of deletion: removing its folder removes everything it owned.

## 8. Testing shape

Three kinds of test, in order of preference. Read `TestingReference.md` in the NnArgumentParser skill for the API before writing any of them.

1. **Interpreters as pure functions.** Build the facts (or findings, requirements, answers) by hand, call `interpret`, assert on the result or the thrown `<Feature>Error`. No mocks needed.
2. **Workflow wiring.** Build a `<Feature>Workflow` with `MockCommandInspector`, `MockCommandCollector`, and `MockCommandResolver` around the real interpreters, run it, and assert on `capturedRequirements` and `capturedContexts`.
3. **Whole command.** Drive the root with `try await testRun`, passing the test factory vending mock infrastructure, the command's arguments, and the interaction mode explicitly; assert on the captured `NnOutput` output or on what the mocks recorded. Progress written through a display is not captured; the test factory vends `MockProgressDisplay` from the `ConsoleProgressTesting` product, and the test asserts on what it recorded (`trackedTitles`, each activity's `updates` and `outcome`, `notes`). A use-case test injects its own `MockProgressDisplay` the same way. Interrupt cleanups are tested through `testRunCapturingCleanups`, which returns them without running them; run them and assert `terminalWasRestored` on the factory's display.

Tests live in `Tests/<tool>Tests/`, mirroring the path below `Sources/<tool>/`: `Tests/<tool>Tests/Features/<Feature>/`, or `Tests/<tool>Tests/Features/<Group>/<Feature>/` for a grouped command, with the same role subfolders. The test factory and any shared helper live in `Tests/<tool>Tests/Shared/`. Test code may use default parameter values; production code may not.

Tests are written when the request asks for them. A workflow step that mentions tests is not permission to add them.
