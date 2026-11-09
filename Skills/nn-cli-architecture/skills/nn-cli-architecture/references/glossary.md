# Glossary

The Structural terms of this architecture. Each means only what it says here, and a tool doesn't reuse it for anything else.

## Pipeline

| Term | Name in code | Meaning |
| --- | --- | --- |
| Workflow | `<Feature>Workflow` | The five stages of one command, with no behavior of its own |
| Inspector | `<Feature>Inspector` | Gathers facts and changes nothing the user owns |
| Facts | `<Feature>Inspector.Facts` | What the inspector found |
| Interpreter | `<Feature>InspectionInterpreter`, `<Feature>CollectionInterpreter` | Reads what's known and decides what it means. All decisions live here |
| Findings | `<Feature>Findings` | What inspection settled |
| Requirements | `<Feature>Requirements` | One entry per input, known or still needed |
| Collector | `<Feature>Collector` | Asks for what's still unknown, and only sequences prompters |
| Prompter | `<Value>Prompter` | Owns one question and its immediate follow-ups |
| Answers | `<Feature>Collector.Answers` | Only what was actually asked |
| Brief | `<Feature>Performer.Brief` | Everything settled that the performer needs |
| Performer | `<Feature>Performer` | Carries out the work described by the brief |
| Use case | Agent noun, such as `TagCreator` | A unit of work the performer calls |
| Results | `<Feature>Performer.Results` | What the work produced |
| Results formatter | `<Feature>ResultsFormatter` | Turns results into the final report |

## Command side

| Term | Name in code | Meaning |
| --- | --- | --- |
| Options | `<Feature>Options` | The command's flags |
| Input | `<Feature>Input` | Those flags as plain values |
| Settings | `<Concern>Settings` | What's derived from flags that several commands share |
| Assembly | `<Group>Assembly` | Builds what sibling commands share |
| Command context | `CommandContext` | The bundle of primitives from the factory |

## Helpers and roles

| Term | Name in code | Meaning |
| --- | --- | --- |
| Detector | `<Field>Detector` | Works out one value, from facts or by reading the outside world |
| Resolver | `<Name>Resolver` | Settles one value from what's already known |
| Validator | `<Name>Validator` | Checks a value and throws when it's bad |
| Classifier | `<Name>Classifier` | Sorts something into one of a set of cases |
| Builder | `<Name>Builder` | Assembles a value from settled parts |
| Store | `<Name>Store` | Fronts persisted state |
| Client | `<Name>Client` | Fronts an outside tool or service |

## Word rules

- **Command** appears only on things about the command as a whole.
- **Resolve** means settling a value, as in `resolve(answer:)`.
- **Select** is only for something a person picks.
- **Helper** is only the collective word for the helpers in the table above, as in `add-helper` and `HelperRole`. It never names a job, as `DateHelper` would.
- **Context** means only the dependency bundle.
- **Agents keep no state.** A command runs once and exits, so stages and use cases hold only their collaborators.
