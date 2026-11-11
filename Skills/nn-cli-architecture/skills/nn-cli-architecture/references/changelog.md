# Changelog

Every change to this skill's rules or to what `nnscaffold` generates gets an entry here, written under `## [Unreleased]` in the same change. Before each release, the Unreleased entries are moved by hand into a section headed with the release version. [Workflows/catch-up.md](../Workflows/catch-up.md) reads the sections newer than a tool's marker and turns them into a refactor list for that tool.

Sections are newest first. Never edit a released section except to fix a wrong **Behind when**.

## Entry format

```
### <short title>

- **Changed**: one or two sentences on what the rule or generated output now says or produces, and what it said or produced before.
- **Rules**: where the current rule lives (a `SKILL.md` universal or stage rule, a Don't list item, a reference file and section), or "none".
- **Generated output**: which `nnscaffold` command now generates something different, and which files, or "none".
- **Behind when**: what an existing tool shows when it predates this entry (a type, a file, a call, a missing registration), concrete enough to search for.
```

An entry that relaxes a rule says so under **Changed**; its **Behind when** names what a tool may now simplify, or "nothing to change". A later entry that replaces an earlier one says which entry it replaces.

## [Unreleased]

## [0.12.1]

### Don't item 14 flags the mode flag, not every multi-stage flag

- **Changed**: Don't list item 14 now flags a mode kept as a flag (one that changes which inspector runs or which questions are asked), however many stages read it; switching on it in several stages is the usual sign. A value flag such as `--force` may be read by every stage its one step touches. Before, it flagged any enum or Bool switched on in more than one stage, which included value flags universal rule 14 allows, and missed a mode flag read in one place. Relaxes the item for value flags.
- **Rules**: `SKILL.md` Don't list item 14.
- **Generated output**: none.
- **Behind when**: a flag that swaps the inspector or the set of questions, even when only `makeWorkflow()` or one stage reads it.

### A failure partway through perform says where things stand

- **Changed**: the Perform stage rule now says that when a step fails after an earlier step changed something, the performer or coordinating use case catches the error and throws a `<Feature>Error` case naming what failed (the underlying message as a `String`), what finished, what didn't, and what to run next. A use case in the executable throws `<Feature>Error` cases (a shared one, the error in its closest `Shared/Workflow/`); only a library use case has its own error type. Flow files leave these failures out of the stops section. Before, the rule didn't say who reports such a failure or how.
- **Rules**: `SKILL.md`, the Perform stage rule and Don't list item 48; `references/flow-format.md`, Rules; `references/architecture.md` section 6 already points to **Say where things stand** in `nn-architecture`.
- **Generated output**: none.
- **Behind when**: a performer or coordinating use case with two or more steps that change something, where a later step's error propagates uncaught (no `catch` that throws a `<Feature>Error` case naming the finished steps), or a flow file that lists such a failure under its stops.

### Model rules stay on the model, and shared state changes atomically

- **Changed**: interpreters make the command's decisions, and a rule about a model's own data lives on the model. Before, the stage rules and glossary said every decision lives in the interpreters. Adds the shared state rule: state another process may write between inspect and perform is changed through one atomic store update (read, apply the model's rule, write, under a lock), the rule returns the outcome for the performer to report, and the inspector's snapshot of it is for preconditions and the report only. Before, the skill had no rule for such state.
- **Rules**: `SKILL.md`, the stage rules opening, the shared state rule, and Don't list item 47; `references/glossary.md`, the Interpreter row.
- **Generated output**: none.
- **Behind when**: a store whose file another process also writes (a lock, queue, or shared state file) that is read and then written as separate steps (in the inspector and performer, or `load()` then `save()` in a use case), with no atomic `update(_:)` on the store.

## [0.12.0]

### A spec that names a factory type needs imports

- **Changed**: `add-command` and `add-use-case` now stop when a spec names `Shell`, `FileSystem`, `Directory`, `CommandLinePicker`, or `ProgressDisplay` and its `imports` is empty. Before, the spec passed and the generated files didn't compile.
- **Rules**: `Workflows/add-command.md`, the type rule in step 2b and **Adding a use case to an existing command**.
- **Generated output**: none. `cli add-command --spec` and `cli add-use-case` refuse the spec before writing.
- **Behind when**: nothing to change in the tool. A saved spec with no `imports` that names one of these types stops until the modules are listed.

### Several rules relaxed to match how tools are built

- **Changed**: relaxes these rules. `<Feature>Collector.swift` may sit beside `Prompters/`, the `isX` ban covers only stored Bool properties, a module builds its own stateless collaborators, a Detector may read the outside world, and an Options type keeps `validate()` in its body. Also adds Resolver and the collective word Helper to the glossary, and says a shared helper's error lives in the closest `Shared/Workflow/`.
- **Rules**: `SKILL.md` Don't list items 33 and 36; `references/architecture.md` sections 6 and 7; `references/style.md`, Type body and Naming; `references/glossary.md`; the `nn-architecture` skill, Dependencies between modules.
- **Generated output**: none.
- **Behind when**: nothing to change.

### Pipeline stages use Inspector, Collector, Performer, and Brief

- **Changed**: the last stage is now the Perform stage. `<Feature>CommandResolver` is `<Feature>Performer`, `resolve(context:)` is `perform(brief:)`, the nested `Context` is `Brief`, and `<Feature>CommandCollector` is `<Feature>Collector`. The protocols and mocks follow NnArgumentParser: `NnInspector`, `NnCollector`, `NnPerformer`, `MockInspector`, `MockCollector`, `MockPerformer`, and `capturedBriefs`. This is a breaking change that needs NnArgumentParser 2.0.0.
- **Rules**: `SKILL.md` stage rules; `references/architecture.md` sections 3, 4, 7, and 8; `references/glossary.md`; `references/style.md`, Naming.
- **Generated output**: `cli add-command` writes the new names, with the performer in `Performer/`. `add-use-case` wires into the performer. `add-command --inspector` only finds inspectors that conform to `NnInspector`, so a tool still on `NnCommandInspector` has no reusable inspectors until it is renamed. `cli init` writes `from: "2.0.0"` for NnArgumentParser in `<tool>/Package.swift`. It wrote `1.3.0` before.
- **Behind when**: the tool's `Package.swift` asks for NnArgumentParser below 2.0.0, or the tool has `NnCommandResolver`, `NnCommandInspector`, `NnCommandCollector`, `CommandResolver`, `CommandCollector`, `resolve(context:)`, a `Resolver/` folder, or, in its tests, `MockCommandResolver`, `MockCommandInspector`, `MockCommandCollector`, or `capturedContexts`.

### Use cases are agent nouns, and the spec names them in full

- **Changed**: a use case is now named as an agent noun with no suffix (`TagCreator`), in `UseCases/TagCreator.swift`. In a spec, `name` is the full type name, the `property` key is gone, and the `context` key is now `brief`. The property is the type name with its whole leading acronym lowercased (`URLFetcher` gives `urlFetcher`). Before, `name` left off a `UseCase` suffix the tool added, and `property` was written by hand.
- **Rules**: `SKILL.md`, the Perform stage rule and Don't list item 29; `references/style.md`, Naming; `Workflows/add-command.md` step 2b and **Adding a use case to an existing command**.
- **Generated output**: `cli add-command --spec` and `cli add-use-case` read the new spec format, refuse a `name` ending in `UseCase`, and write `UseCases/<Name>.swift`.
- **Behind when**: a type or file under `UseCases/` ends in `UseCase`, or a performer holds a use case under a property that isn't its type name in lower camel case. A saved spec with a `property` or `context` key stops with an unknown-key error.

### Follow-up prompters use `--stage perform`

- **Changed**: `cli add-prompter --stage resolve` is now `--stage perform`. Where the prompter goes hasn't changed.
- **Rules**: `SKILL.md`, the Perform stage rule; `references/architecture.md` section 7.
- **Generated output**: `cli add-prompter` takes `--stage collect|perform`.
- **Behind when**: a script or document in the tool passes `--stage resolve`.

### Tools get architecture rules tests

- **Changed**: a generated test file now fails when a tool breaks Don't list items 1, 10 (`print` only), 34, 37, 38, 42, or 43, naming the file and the rule. Before, those items were only checked by review.
- **Rules**: `references/architecture.md` section 8; the items marked Checked in the `SKILL.md` Don't list.
- **Generated output**: `cli init` writes `Tests/<tool>Tests/Main/ArchitectureRulesTests.swift`. The new `cli add-architecture-tests` adds it to an existing tool and replaces it when the rules change.
- **Behind when**: `Tests/<tool>Tests/Main/ArchitectureRulesTests.swift` is missing, or `nnscaffold cli add-architecture-tests --preview` shows it would change.

### Scaffolded type names refuse catch-all words

- **Changed**: `add-use-case`, `add-command`, `add-helper`, and `add-prompter` now stop on a type name that uses Manager, Helper, Handler, Processor, Service, Util, or Utility, and the error points to `nn-architecture` naming rule 6.
- **Rules**: `nn-architecture`, Choosing a name, rule 6; `SKILL.md` Don't list item 46.
- **Generated output**: those four commands refuse the name before writing anything.
- **Behind when**: a type the tool named itself uses one of those words. Glossary terms are exempt.

### The performer coordinates, and shared principles live in `nn-architecture`

- **Changed**: the performer is the coordinator, and a coordinating use case exists only when two commands share the sequence or the sequence runs concurrently. `makeWorkflow()` also builds the use cases one use case gives to another. Performers and use cases are now a fourth kind of test, and a failure after the first change follows **Say where things stand** in `nn-architecture`.
- **Rules**: `SKILL.md`, universal rule 5, the Perform stage rule, and the routing row for `nn-architecture`; `references/architecture.md` sections 6 and 8; the `nn-architecture` skill.
- **Generated output**: none.
- **Behind when**: a use case that only calls other use cases in order, used by one command and not concurrent; a use case that builds another use case; or an error thrown after the first change that doesn't say what was done, what wasn't, and what to do next.

### The Don't lists are marked by how each item is enforced

- **Changed**: each item in the `nn-cli-architecture` and `nn-mcp-architecture` Don't lists is marked Generated, Checked, or Judgment, and `Workflows/review-cli.md` step 4 follows the marks.
- **Rules**: the Don't list in each `SKILL.md`; `Workflows/review-cli.md` step 4.
- **Generated output**: none.
- **Behind when**: nothing to change.

### Generated packages treat warnings as errors

- **Changed**: both packages `cli init` creates now end with a `for target in package.targets` loop that adds `.treatAllWarnings(as: .error)` to every target. Before, warnings didn't fail the build.
- **Rules**: none.
- **Generated output**: `cli init` adds the loop to `<App>Core/Package.swift` and `<tool>/Package.swift`.
- **Behind when**: either `Package.swift` has no `treatAllWarnings(as: .error)`.

### Core Testing libraries re-export their Kit

- **Changed**: `<App>FileTesting` and `<App>ShellTesting` now depend on `<App>FileKit` or `<App>ShellKit` and re-export it next to NnFileTesting or NnShellTesting, so a test importing a Testing library also sees its Kit. Before, they depended on and re-exported only the Nn testing module.
- **Rules**: none.
- **Generated output**: `cli init` adds the Kit target to each Testing library's dependencies in `<App>Core/Package.swift` and an `@_exported import <App>FileKit` or `<App>ShellKit` line to its `Exports.swift`.
- **Behind when**: `<App>Core/Sources/<App>FileTesting/Exports.swift` or `<App>ShellTesting/Exports.swift` doesn't import its Kit.

## [0.11.2] - 2026-11-07

### Follow-up prompters live in `FollowUps/`

- **Changed**: A prompter a use case asks as a follow-up, once the work has produced something, now lives in the feature's `FollowUps/` folder (or `<Group>/Shared/FollowUps/`, `Shared/FollowUps/`) and comes from `nnscaffold cli add-prompter --stage resolve`, which works for passthrough commands too. Before, every prompter lived in `Collector/Prompters/`, so a passthrough command had no place for one, and a collecting command kept its use-case prompters beside the collector's.
- **Rules**: `SKILL.md`, the Resolve stage rule and Don't list item 33; `references/architecture.md` section 7.
- **Generated output**: `cli add-prompter` gains `--stage collect|resolve`; `--stage resolve` writes `<Feature>/FollowUps/<Value>Prompter.swift` and its test in `Tests/<tool>Tests/Features/<Feature>/FollowUps/`. The error for a feature with no `Collector/` now points to `--stage resolve`.
- **Behind when**: a prompter in `Collector/Prompters/` (and its test) that only a use case or resolver calls, not the collector; or a use case that calls the picker directly for a follow-up instead of through a `<Value>Prompter`.

## [0.11.1] - 2026-11-07

### `--yes` approves confirmations in a terminal

- **Changed**: `DefaultContextFactory.makePicker()` now passes the mode's `assumeYes` to the interactive picker, `SwiftPicker(interruptBehavior: .waitForTermination, assumeYes:)`, so `--yes` approves confirmation prompts in a terminal as well. Before, the interactive case ignored `assumeYes`, so `--yes` only took effect in non-interactive runs.
- **Rules**: `references/architecture.md`, the `DefaultContextFactory` row.
- **Generated output**: `cli init` now generates the new interactive case in `Main/DefaultContextFactory.swift` and requires SwiftPickerKit 0.14.0 in the executable's `Package.swift`.
- **Behind when**: `makePicker()` in `DefaultContextFactory.swift` has `case .interactive:` without binding `assumeYes`, or the tool's `Package.swift` requires SwiftPickerKit below 0.14.0.

### Core package requires NnFileKit 0.9.2

- **Changed**: the generated Core package now requires NnFileKit 0.9.2, where `MockFileSystem.directory(at:)` and `MockDirectory.subdirectory(named:)` throw `FileSystemError.directoryNotFound` for a missing directory, matching the real implementations. Before, it required 0.9.1, where those mocks threw undocumented `NSError` codes.
- **Rules**: none.
- **Generated output**: `cli init` now requires NnFileKit 0.9.2 in the Core package's `Package.swift`.
- **Behind when**: the Core package's `Package.swift` requires NnFileKit below 0.9.2, or a test matches `NSError` domain `"MockFileSystem"` code 1 or `"MockDirectory"` code 2 for a missing directory.

## [0.10.0] - 2026-11-03

### Library targets stay free of CLI packages

- **Changed**: a library target now can't depend on NnArgumentParser, ConsoleProgressKit, or SwiftPickerKit, and the rule gives its reason: the target holds logic that isn't tied to a terminal, so it stays reusable and testable without one. Before, only NnArgumentParser was named, with no reason given.
- **Rules**: `references/architecture.md` section 5; `SKILL.md` Don't list item 34.
- **Generated output**: none.
- **Behind when**: a library target in the Core package's `Package.swift` lists `NnArgumentParser`, `ConsoleProgressKit`, or `SwiftPickerKit` as a dependency, or a file under `<App>Core/Sources/` imports one of them.

### Slow inspection is tracked with NnTrackedInspector

- **Changed**: a command whose inspection is slow now shows progress by wrapping its inspector in `NnTrackedInspector` in `makeWorkflow()`, and an inspector never takes the progress display, in any target. A use case that tracks work stays in the executable. Before, inspectors were told not to receive the display, with no reason given and no other way to show progress during inspection.
- **Rules**: `SKILL.md` Inspect stage rule and Don't list item 44 (flow files move to item 45); `references/architecture.md` sections 2, 3, 5, and 8; `Workflows/add-command.md` steps 1 and 4b.
- **Generated output**: `nnscaffold cli init` writes `from: "1.3.0"` for NnArgumentParser in `<tool>/Package.swift`. It wrote `1.1.0` before. `nnscaffold cli add-command --inspector` can reuse an inspector another command wraps in `NnTrackedInspector`, and copies it without the wrapper.
- **Behind when**: an inspector's `init` takes `any ProgressDisplay`, or an inspector that makes network calls has no wrapper in its `<Feature>Workflow`. Raise NnArgumentParser to 1.3.0 in `Package.swift` before wrapping.

### Tools can choose progress colors

- **Changed**: a tool may pass a `ProgressTheme` to `LiveProgressDisplay(theme:)` in `DefaultContextFactory`. This relaxes nothing else: the display is still constructed only there, and `LiveProgressDisplay()` stays the generated default.
- **Rules**: `SKILL.md` universal rule 2; `references/architecture.md` section 2.
- **Generated output**: `nnscaffold cli init` writes `from: "0.7.0"` for ConsoleProgressKit in `<tool>/Package.swift`. It wrote `0.6.0` before.
- **Behind when**: nothing to change.

## [0.9.0] - 2026-11-01

### Commands get flow files

- **Changed**: every leaf command now has a plain-English flow file at `<tool>/Flows/<command path>.md`, in the format of `references/flow-format.md`. A generated `FlowFilesTests` test fails when a command has no flow file or a flow file has no command. Add-command writes the flow file as its new step 6. Before, commands had no written explanation outside their code.
- **Rules**: `references/flow-format.md`; `references/architecture.md` sections 7 and 8; `Workflows/add-command.md` step 6; `Workflows/review-cli.md` step 4; `SKILL.md` Don't list item 45 and **Completion**.
- **Generated output**: `nnscaffold cli init` now writes `Tests/<tool>Tests/Main/FlowFilesTests.swift`. A new command, `nnscaffold cli add-flow-tests`, writes the same file into an existing tool.
- **Behind when**: the tool has no `Tests/<tool>Tests/Main/FlowFilesTests.swift`, or no `Flows/` folder. The fix is **Adding flow files to an existing tool** in `references/flow-format.md`: run `nnscaffold cli add-flow-tests`, then write every flow file `swift test` lists, in one sitting.

### add-command reuses only shared facts factories

- **Changed**: with `--inspector`, `add-command` now looks for the inspector's facts factory only in `Tests/<tool>Tests/Shared/`, and writes a new `make<Feature>Facts()` there when none is found. Before, it searched every test file, so it could pick a `private` helper inside another feature's tests, and the generated workflow test failed to compile.
- **Rules**: `Workflows/add-command.md` **4a. Generate** (the workflow tests call the inspector's existing facts factory from `Tests/<tool>Tests/Shared/`).
- **Generated output**: `nnscaffold cli add-command` with `--inspector`: `<Feature>WorkflowTests.swift`, and `Shared/TestFactory+<Feature>.swift` when no shared factory exists.
- **Behind when**: a `<Feature>WorkflowTests.swift` calls a facts factory that isn't declared in `Tests/<tool>Tests/Shared/`, which shows up as `cannot find '<name>' in scope` when the tests build.

## [0.8.2] - 2026-11-01

### add-use-case builds a shared dependency once

- **Changed**: `add-use-case` now builds a dependency once when its `name` and `from` match an argument already in `makeWorkflow()` or another new use case's. It reuses an existing `let <name> = <from>`, or turns the inline argument into a `let` before `return .init(` and passes `<name>` everywhere. Before, it passed its own copy of the expression, which broke universal rule 5. This replaces the "`add-use-case` doesn't hoist" note in **Shared dependencies built once**.
- **Rules**: `SKILL.md` universal rule 5; `Workflows/add-command.md` **Adding a use case to an existing command**.
- **Generated output**: `nnscaffold cli add-use-case` edits `Command/<Feature>.swift` (the `let`, the rewritten inline arguments, and the new construction), or prints those lines when the command doesn't have the generated shape.
- **Behind when**: a `makeWorkflow()` that builds the same expression in two `from` arguments after an `add-use-case` run (for example `GitClient(...)` passed to both the inspector and a use case). Move it into one `let` and pass the name.

### Types in any Core target are recognized

- **Changed**: The undefined-type check now reads every library target in the Core package. Before, it read only the targets exported as products (the Kits), so a type declared in another Core target had to be listed in `knownTypes`.
- **Rules**: `Workflows/add-command.md` step 2b (`knownTypes`).
- **Generated output**: none.
- **Behind when**: `knownTypes` lists a type declared in the Core package. Remove it.

## [0.8.1] - 2026-11-01

### Helpers import only what they use

- **Changed**: Helper files no longer import `NnArgumentParser`. A spec helper gets the spec's `imports` only when its parameters or return type name a type that isn't a Swift or Foundation built-in or generated by the spec. Its example tests follow the same rule. Before, every helper imported `NnArgumentParser` and every spec module.
- **Rules**: `Workflows/add-command.md` step 2b (`imports`) and the `add-helper` description.
- **Generated output**: `nnscaffold cli add-helper` and `nnscaffold cli add-command --spec` write helpers in `Interpreters/<Stage>/` and their tests with fewer imports.
- **Behind when**: a helper that imports `NnArgumentParser` or a Kit it doesn't use. Nothing to change.

### Shared dependencies built once

- **Changed**: When the same `from` expression appears in more than one inspector or use-case dependency of an `add-command` spec, each with the same `name`, `makeWorkflow()` now builds it once as `let <name> = <from>` and passes `<name>` to each stage. Before, each stage got its own copy of the expression, which broke universal rule 5.
- **Rules**: `SKILL.md` universal rule 5; `Workflows/add-command.md` step 2b (`useCases`).
- **Generated output**: `nnscaffold cli add-command --spec` writes the `let` into `makeWorkflow()` in `Command/<Feature>.swift` and passes the local value in the inspector and resolver constructions. `add-use-case` doesn't hoist into an existing `makeWorkflow()`.
- **Behind when**: a `makeWorkflow()` that builds the same expression in two `from` arguments (for example `GitClient(...)` passed to both the inspector and a use case).

### Command file imports the spec's modules

- **Changed**: A command file whose `makeWorkflow()` builds a dependency from a `from` expression now imports every module in the spec's `imports`. `add-use-case` adds any of those imports that the command file is missing, or prints them when the file doesn't have the generated shape. Before, the command imported only `NnArgumentParser`, Foundation, and the inspector's Kit, so a `from` that named a type from a Kit didn't compile until the import was added by hand.
- **Rules**: `Workflows/add-command.md` step 2b (`imports`) and **Adding a use case to an existing command**.
- **Generated output**: `nnscaffold cli add-command --spec` writes the spec's imports into `Command/<Feature>.swift`; `nnscaffold cli add-use-case` edits that file's imports.
- **Behind when**: a command file that builds a `from` type and imports its module by hand. Nothing to change.

## [0.8.0] - 2026-11-01

### New command: `add-use-case`

- **Changed**: `nnscaffold cli add-use-case --feature <Feature> --spec <file.json>` adds the use cases listed in a spec file to an existing command. Each use case gets its source file and test file, and the resolver, `makeWorkflow()`, and the resolver tests' `makeSUT` are wired when they still have the generated shape, adding `let context = <Root>.makeContext()` to `makeWorkflow()` when a new dependency reads `context` and the command has no such line. When they don't have that shape, the lines to add are printed instead, the context line among them when it's needed. Before, a use case added after `add-command` was written by hand.
- **Rules**: `SKILL.md` Routing; `Workflows/add-command.md` step 4b and **Adding a use case to an existing command**.
- **Generated output**: `nnscaffold cli add-use-case` writes `UseCases/<Name>UseCase.swift` and its test, and edits `<Feature>CommandResolver.swift`, the command file, and `<Feature>CommandResolverTests.swift`, or prints the lines for them.
- **Behind when**: a use case written by hand after the command was generated. Nothing to change; the next use case for that command can come from `add-use-case`.

### Spec: option groups, test values, inspector dependencies, existing prompters

- **Changed**: The spec can now include a shared `<Concern>Options` (`optionGroups`), give a fact or Input field a test expression when its type has no empty value (`testValues`), give a new feature inspector its dependencies (`inspector.dependencies`), and ask a requirement through a prompter the tool already has (`"prompter": { "existing", "method" }`). Before, each was a hand edit after generating.
- **Rules**: `Workflows/add-command.md` steps 1, 2, and 2b.
- **Generated output**: `nnscaffold cli add-command --spec` writes the `@OptionGroup` and its Input field, the inspector's stored dependencies and `init` with their construction in `makeWorkflow()` and the inspector test's `makeSUT`, and the existing prompter's collector property and ask. It imports `Foundation` in the command when a dependency's `from` builds a `Date`, `URL`, `UUID`, or `Data`.
- **Behind when**: nothing to change; a command generated before this works as it is.

### Helper example tests start disabled

- **Changed**: The parameterized tests generated from a spec helper's `examples` now start with a `.disabled("nnscaffold: implement …, then remove this trait")` trait. Before, they ran against the helper's `fatalError` stub, which crashed the whole test run.
- **Rules**: `Workflows/add-command.md` step 2b (`examples`).
- **Generated output**: `nnscaffold cli add-command --spec`, the `<Helper>Tests.swift` file of each helper with examples.
- **Behind when**: nothing to change once the helper body is written; a helper test with examples and no trait already runs.

### Helper files import the spec's modules

- **Changed**: A spec helper's source file, and its test file when it has examples, now import the spec's `imports`. Before, they imported only `NnArgumentParser` and compiled only when another file in the module happened to import the same modules.
- **Rules**: `Workflows/add-command.md` step 2b (`imports`).
- **Generated output**: `nnscaffold cli add-command --spec`, each helper's source and test file.
- **Behind when**: a helper in `Interpreters/` names a type from a module its file doesn't import.

### Formatter stubs bind each case's summary

- **Changed**: In a spec's results formatter, each case with a summary now binds it, as in `case .created(let summary):`, and the stub's `fatalError` message names it. Before, the arms didn't bind the summary, so writing the message meant changing the `case` line too.
- **Rules**: `Workflows/add-command.md` step 2b (`results`).
- **Generated output**: `nnscaffold cli add-command --spec`, `<Feature>ResultsFormatter.swift`.
- **Behind when**: nothing to change; an arm written before this binds the summary by hand.

### Helper example values follow the parameter type

- **Changed**: In a spec helper's `examples`, a JSON string given for a `String` or `String?` parameter is written as a string literal, and `null` is written as `nil`. Any other string is still written exactly as given, as Swift source. Before, a `String` value needed escaped quotes inside the JSON string, and `null` was rejected. A helper with examples can no longer have a parameter named `expected`, because that name clashes with the generated test's tuple label.
- **Rules**: `Workflows/add-command.md` step 2b (`examples`).
- **Generated output**: `nnscaffold cli add-command --spec`, the examples in each `<Helper>Tests.swift`.
- **Behind when**: nothing to change.

### The spec is the plan

- **Changed**: The add-command workflow writes each decision once, in the spec: types, signatures, use cases, results, error messages, and helper examples. The prose plan, called the behavior notes, keeps only what the spec can't express: interpreter decisions, the inspector's reads, use-case step bodies, and the formatter's wording. The approval step shows the spec and the `--preview` file list. Before, a stage table and a file table repeated what the spec then said again.
- **Rules**: `Workflows/add-command.md` steps 1, 2a, 2b, and 4b; `Workflows/new-cli-project.md`.
- **Generated output**: none.
- **Behind when**: nothing to change in a tool.

### Spec: helper examples become tests

- **Changed**: A spec helper can list `examples`. Each one has a `given` value for every parameter, plus exactly one of `expect` or `throws`. They become parameterized tests in the helper's test file, one for returned values and one for thrown errors. Before, a helper's tests were written by hand from the plan.
- **Rules**: `Workflows/add-command.md` steps 2b (`examples`) and 4b (helper bodies are written against their example tests).
- **Generated output**: `nnscaffold cli add-command --spec`, each `<Helper>Tests.swift` whose helper has examples.
- **Behind when**: nothing to change.

### Spec: Context, Results, optional requirements, and error messages

- **Changed**: The spec's `context` generates the resolver's `Context`. Its `results` generate `Results`, with a `<Case>Summary` struct for each case that has a summary, and a formatter that switches over every case. A requirement marked `"optional": true` becomes `NnRequirement<…>?`, asked with `?.ask`. An error's `message`, with `{value}` tokens, generates its `errorDescription`. Before, all of these were written by hand. A passthrough command can't have `context`.
- **Rules**: `Workflows/add-command.md` step 2b (`context`, `results`, `requirements`, `errors`).
- **Generated output**: `nnscaffold cli add-command --spec`, which changes `<Feature>CommandResolver.swift`, `<Feature>ResultsFormatter.swift`, `<Feature>Requirements.swift`, `<Feature>Error.swift`, and the workflow tests.
- **Behind when**: nothing to change.

### Spec: use cases

- **Changed**: The spec's `useCases` generate each use case's file and test file in `UseCases/`, plus the resolver's stored properties and `init`, their construction in `makeWorkflow()`, and the resolver tests' `makeSUT`. Each use case has a `name`, a `property`, its `dependencies` as `{ name, type, from }`, and a `method` signature. Before, use cases and their wiring were written by hand after generating.
- **Rules**: `Workflows/add-command.md` steps 2b (`useCases`) and 4b.
- **Generated output**: `nnscaffold cli add-command --spec`, which writes `UseCases/<Name>UseCase.swift` and its test, and changes `<Feature>CommandResolver.swift`, the command file, and `<Feature>CommandResolverTests.swift`.
- **Behind when**: nothing to change.

### Helpers get their real signatures

- **Changed**: `add-helper` takes `--param name:Type` (repeatable), `--returns`, and `--throws`, and writes the stub with that exact signature. The spec's `helpers` section does the same for each helper, placing it as `add-helper` does. Before, `--value-type` gave every helper one `String`-style parameter, and the signature was then edited by hand. A classifier or builder needs `--returns`.
- **Rules**: `Workflows/add-command.md` steps 2b (`helpers`) and 4b (`add-helper`).
- **Generated output**: `nnscaffold cli add-helper` and `nnscaffold cli add-command --spec`, each `<Name><Role>.swift` and its test.
- **Behind when**: nothing to change; a helper whose signature was edited by hand stays as it is.

### Spec: models, knownTypes, and the undefined-type check

- **Changed**: The spec's `models` generate structs and enums in `Models/`. `knownTypes` names types from other packages. A type the spec names that is neither built in, generated, declared in the tool or its Core package, nor listed in `knownTypes` now stops the run before anything is written. Before, model files were written by hand, and a misspelled type surfaced only as a build error.
- **Rules**: `Workflows/add-command.md` step 2b (`models`, `knownTypes`).
- **Generated output**: `nnscaffold cli add-command --spec`, which writes `Models/<Name>.swift`.
- **Behind when**: nothing to change.

### Generated tests hold no placeholder test

- **Changed**: Generated test files hold the test struct and its `makeSUT`, with no placeholder test. The report lists them under `Tests to write:`, so `swift test` passes before any test is written. Before, each file had a `` `Replace with a real test` `` test that recorded an issue, so the suite stayed red until every placeholder was replaced.
- **Rules**: `Workflows/add-command.md` steps 4a and 5.
- **Generated output**: `nnscaffold cli add-command`, `cli add-helper`, and `cli add-prompter`, every test file they write.
- **Behind when**: a test named `` `Replace with a real test` `` that calls `Issue.record`. Replace it with real tests, or delete it.

### Changelog and catch-up workflow added

- **Changed**: The skill now records its changes here, and a tool records the skill version it was last brought up to in a marker file. Nothing about the architecture changed.
- **Rules**: none.
- **Generated output**: none.
- **Behind when**: the tool has no marker file. Its catch-up is a full review against the current rules.
