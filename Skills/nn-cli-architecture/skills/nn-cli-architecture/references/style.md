# File Style

How a Swift source file in the tool is laid out and named. Read completely before writing the first file of a task. When this file and `references/architecture.md` disagree on what a type does, `architecture.md` wins; on how a file looks, this file wins.

## Header and comments

- A Swift file opens with the standard header (file name, module name, and the "Created by" line with author and date) when the project's Swift files use one, and with no header when they do not. Match the project. `Package.swift` is exempt.
- Nothing is written above a declaration except `// MARK:` and `// TODO:`. No `///` documentation, no explanatory `//` comments. The reasoning behind a decision goes in the reply or the plan, not in the source.

## Type body and extensions

- The type body holds stored properties and `init`, nothing else. Methods, computed properties, nested types, and conformances live in extensions below, each under a MARK.
- Three exemptions, because the whole body is a framework contract: a `ParsableCommand` keeps `configuration`, its `@OptionGroup` and flag properties, and `makeWorkflow()` / `handleResults` in the body; `DefaultContextFactory` keeps its `make*` methods in the body; an Options type keeps `validate()` in the body.
- A caseless helper enum keeps its public static API in the body and its private static helpers in a `private extension`.
- An implemented type whose body would be completely empty (no stored properties and no `init`) is not declared as `struct X { }` with its code in extensions. Its declaration adopts its protocol and its body holds that surface. Private helpers, nested types, and further conformances stay in extensions below under their MARKs. A type with any `init`, including a `public init() { }`, keeps the usual split. Generated stubs (`fatalError("nnscaffold: implement …")`), generated test structs with no tests yet, and payloads with no fields yet stay as generated until they are implemented.

```swift
struct TagReleaseCollectionInterpreter: NnCollectionInterpreter {
    func interpret(findings: TagReleaseFindings, requirements: TagReleaseRequirements, answers: TagReleaseCollector.Answers) throws -> TagReleasePerformer.Brief {
        …
    }
}

// MARK: - Private Methods
private extension TagReleaseCollectionInterpreter {
    …
}
```

## MARK order

1. The public surface (`// MARK: - Inspect`, `// MARK: - Build`, or the protocol name when the surface is a conformance). Absent when the surface sits in the body of an otherwise empty type.
2. Behaviour-named private extensions, when the private methods fall into groups.
3. `// MARK: - Private Methods` for the rest.
4. `// MARK: - Dependencies` for nested satellite types.
5. One extension per protocol conformance, MARKed with the protocol's name (`// MARK: - NnCollector`, `// MARK: - LocalizedError`).

Omit a MARK the type does not need.

## Nesting and files

- Satellite types that serve one owner (its `Answers`, `Brief`, `Results`, an option enum) nest inside the owner, in the owner's own file under `// MARK: - Dependencies`.
- A type two owners use gets its own file, named for the type.
- Do not group unrelated types in one file. A file's name says what is in it.
- Step sequences are modeled as enums with one case per step, not as nested functions or chains of closures. No nested functions: a helper goes in a private extension.

## Expressions

- `return` is explicit, including in single-expression bodies.
- `guard` for early exits; `if` for genuine branches.
- A closure passed as a call's last argument may be written as a trailing closure.
- Long initializer calls put one argument per line.

## Naming

The terms in this table are defined in [glossary.md](glossary.md). A name a project chooses follows the naming rules in the `nn-architecture` skill.

| Thing | Name |
| --- | --- |
| Stage types | `<Feature>Inspector`, `<Feature>InspectionInterpreter`, `<Feature>Collector`, `<Feature>CollectionInterpreter`, `<Feature>Performer`, `<Feature>Workflow` |
| Command-side types | `<Feature>Options`, `<Feature>Input`, `<Feature>ResultsFormatter`, `<Feature>Error` |
| Payload models | `<Feature>Findings`, `<Feature>Requirements` |
| One question's prompts | `<Value>Prompter` |
| Working out one input from facts | `<Field>Detector` with `static func detect(...)` |
| Use cases | An agent noun named for the job, such as `ReleasePublisher` or `PackageCompiler`, with no `UseCase` suffix and never a glossary term. The property holding one is the type name with its whole leading acronym lowercased (`releasePublisher`, `urlFetcher`) |
| Booleans | stored properties are subject first: `fileExists`, `branchIsClean`, `remoteWasFetched`, never with an `isX` prefix. Computed properties and functions may use `isX`. |
| "Select" | only for something a person picks. A computed choice is detected, built, or resolved. |

## Parameters and models

- Default parameter values follow universal rule 10 in `SKILL.md`. Test targets and `*Testing` targets may default parameters.
- Payload and model types are `Sendable` structs or enums; `Equatable` when compared in tests or logic; `Codable` when persisted.
- A `public` model declares an explicit `public init` listing every field; the synthesized initializer is internal and would deny construction across packages.
