# Flow File Format

A flow file tells someone what a command does before they run it: what happens, what it asks, and when it stops. It is written for that reader alone. It is not a map of the code (that is one line at the bottom) or a list of every message (that is the error types).

Every leaf command has one; a group does not. The generated `FlowFilesTests` test fails when a command has no flow file or a flow file has no command, so a missing file shows up in `swift test`. Whether a file is accurate is checked by reading it against the code (add-command step 6, and `Workflows/review-cli.md`).

## Location

`<tool>/Flows/<command path>.md`, at the package root beside `Sources/` and `Tests/`, never inside `Sources/` (SwiftPM warns about unhandled files there). The path mirrors how the command is typed, without the tool name: `add-target` is `Flows/add-target.md`, and `cli add-command` is `Flows/cli/add-command.md`. A group is a folder, never a file.

## Template

```markdown
# <command path>

<One sentence: what the command does, in the user's words.>

## What happens

1. <The first thing the user would notice, in plain words.>
2. <…>

<One line on what --preview or a dry run changes, and on any flag that answers no question (such as --header), when the command has them.>

## Skipping the questions

| Question | Pass |
| --- | --- |
| <what it asks> | <the argument or flag that answers it> |

<When a question is skipped without a flag, and what happens without a terminal.>

## It stops, before changing anything, when

- <A situation the user can cause, and what to do about it.>

<At most two lines on behavior that would surprise the user.>

## Code

`<Feature>Options` → `<Inspector>` → `<Feature>InspectionInterpreter` → `<Feature>Collector` → `<Feature>CollectionInterpreter` → `<Feature>Performer` (`<Use case>`, …) → `<Feature>ResultsFormatter`
```

## Rules

- **Write for the person about to run the command.** Each line answers "what will this do?", "what will it ask?", or "why did it stop?". Leave out anything that doesn't change what they do or expect.
- **What happens** is 4 to 8 numbered steps in the order the user experiences them: what it looks at, what it asks, what it checks, what it writes or runs, and what it prints. Use plain words, not type names. Name a file the command creates or edits when the user will look for it.
- **Skipping the questions** lists each question with the argument or flag that answers it. Leave the section out when the command asks nothing. A flag that answers no question (`--header`, `--preview`) goes in the line under What happens instead.
- **It stops** lists the blocks a user can cause, each with its reason or its fix in plain words. Don't quote the message, and don't list failures that only a broken project could cause: a missing or second `@main` root, a group file with no `subcommands: [` list or several, a file in an unexpected shape, a manifest the tool can't parse. A failure after the first change is not a stop, so its `<Feature>Error` case stays out of this section.
- **Surprises** get at most two lines, after the stops: an order that matters (a typed answer checked only after every question), or something done beyond the obvious.
- **Hard to explain may mean a bug.** If a step can't be explained plainly, or the code does something a user wouldn't expect, tell the user it may be a bug. Don't document around it, and don't change code to fix it while writing the flow file.
- **Type names appear only in the Code line**: the stage types in pipeline order, with the performer's use cases in parentheses. Leave out a stage the command doesn't have (`<Feature>Options` without a `validate()`, the collector and collection interpreter for a passthrough command). A shared inspector keeps its own name, so the line can start with `CommandLineToolInspector` rather than a `<Feature>Inspector`.
- **No cross-references between steps,** no tables beyond the questions table, no planning notes, no test lists.
- **Aim for 25 to 50 lines.** A longer file usually lists detail the reader doesn't need.
- **No em dashes.** Use a colon, a comma, or parentheses.

## Tracing a command into its flow file

Read the code before writing, so every line is true:

| Section | Read |
| --- | --- |
| Summary, What happens | the command's `configuration`, the inspector, the performer and its use cases, and the formatter |
| Skipping the questions | the collector, each prompter's `flagHint` and skip conditions, and `<Feature>Options` |
| It stops | `<Feature>Options.validate()`, the inspection interpreter, `<Feature>Error`, and the collection interpreter |
| Code | the feature folder's stage types and the performer's use cases |

## Example

A collecting command, illustrative only:

```markdown
# tag-release

Tags the current commit with a version.

## What happens

1. Checks that you're in a git repository with no uncommitted changes.
2. Finds the latest version tag.
3. Asks for the new version, offering the next major, minor, and patch.
4. Checks that the version is higher than the latest tag.
5. Creates the tag on the current commit and prints it.

With `--dry-run`, it prints the tag it would create and creates nothing.

## Skipping the questions

| Question | Pass |
| --- | --- |
| Version | the version, as the first argument |

Without a terminal and without a version argument, it stops and asks you to pass one.

## It stops, before changing anything, when

- You aren't in a git repository.
- There are uncommitted changes: commit or stash them first.
- The version isn't higher than the latest tag.

A version typed at the prompt is checked only after you enter it, and isn't asked for again when it fails.

## Code

`TagReleaseInspector` → `TagReleaseInspectionInterpreter` → `TagReleaseCollector` → `TagReleaseCollectionInterpreter` → `TagReleasePerformer` (`TagCreator`) → `TagReleaseResultsFormatter`
```

## Adding flow files to an existing tool

A tool created before flow files existed has no `FlowFilesTests.swift` and no `Flows/` folder. Finish these steps in one sitting: once the test is added, the tool's whole `swift test` fails until every flow file exists.

1. Refresh this skill in the project so these rules are current (`/sync-skill-refs`, or update the plugin).
2. From the folder holding the `.xcodeproj`, run `nnscaffold cli add-flow-tests` (add `--header` when the project's Swift files have headers). Never write the test by hand.
3. Run `swift test` in the tool's package. The failure lists every `Flows/<path>.md` to write.
4. Write each one by tracing its command, following this file.
5. Run `swift test` again: both flow tests pass. Commit the test, the flow files, and the skill refresh together.
