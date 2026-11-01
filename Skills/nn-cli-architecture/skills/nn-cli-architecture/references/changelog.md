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
