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
