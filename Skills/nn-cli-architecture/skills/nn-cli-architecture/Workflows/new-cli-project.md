# Workflow: New CLI Project

Use when no `Package.swift` with an executable target depending on NnArgumentParser exists, or the request is to start a tool from nothing. Produces an `<App>Core` package, the tool's package with `Main/`, and the first command. Every file comes from `nnscaffold`; none of them is written by hand.

## 1. Check the project folder

The working directory must hold exactly one `.xcodeproj`. Its name is the app name, and the tool is created beside it.

If there is no `.xcodeproj`, stop the workflow. Tell the user that a new tool is created beside an app's Xcode project, so they should run it from the folder that holds the `.xcodeproj`. Create no files.

## 2. Settle the inputs

From the request, fix:

- the tool name: lowercase words of letters and digits, each starting with a letter, joined by single hyphens, such as `nnex` or `fetch-tool-cli`. It becomes the executable product and the package folder. The root type `<Root>` is its PascalCase form (`FetchToolCli`), and tests import the module with hyphens turned into underscores (`@testable import fetch_tool_cli`).
- the abstract: one line for the root command's `--help`.
- the first command, planned through steps 1 to 2a of `Workflows/add-command.md`: its spec and behavior notes. Its `--preview` file list comes after step 3 creates the tool.

Ask only for what the request leaves open.

## 3. Create the tool

From the folder holding the `.xcodeproj`, run with every value passed:

```
nnscaffold cli init <tool> --abstract "<abstract>" --non-interactive [--header]
```

Files get no header comment unless you pass `--header`. Pass it when the user wants headers, or when the project's existing Swift files have them. With it, every file gets a header signed with the git `user.name`, and the command stops if none is set.

It creates and registers in the Xcode project, without linking either to the app target:

- `<App>Core/`: the `<App>FileKit` and `<App>ShellKit` libraries, each re-exporting NnFileKit or NnShellKit, and the `<App>FileTesting` and `<App>ShellTesting` libraries for the tool's tests, each depending on its Kit and re-exporting both NnFileTesting or NnShellTesting and that Kit.
- `<tool>/`: one executable target with `Main/<Root>.swift`, `Main/DefaultContextFactory.swift`, `Main/CommandContext.swift`, and `Main/ProgressAwareOutputWriter.swift`. The root vends a shell, a file system, a picker, and a progress display, registers `ProgressAwareOutputWriter` and the display's `restoreTerminal()`, registers `ChildProcesses.terminateAll(gracePeriod: 2)` as an interrupt cleanup in `makeContext()`, and has no subcommands yet. A `<tool>Tests` target holds `Tests/<tool>Tests/Shared/MockContextFactory.swift`, the test factory conforming to `ContextFactory`, `Tests/<tool>Tests/Shared/<Root>TestRunner.swift`, which drives the root through `testRun`, `Tests/<tool>Tests/Main/FlowFilesTests.swift`, which fails when a command has no flow file in `Flows/` (`references/flow-format.md`), and `Tests/<tool>Tests/Main/ArchitectureRulesTests.swift`, which fails when the tool breaks a Don't list item marked Checked. It also writes the tool's architecture marker, `<tool>/.nn-cli-architecture`, holding the nnscaffold version (which is this skill's version), so a tool created now starts up to date; `Workflows/catch-up.md` reads it later.

Both `Package.swift` files end with a `for target in package.targets` loop that adds `.treatAllWarnings(as: .error)` to every target, so a warning fails the build.

It then installs the tool's skills at project scope, into the project's `.claude/settings.json`: NnArgumentParser, NnFileKit, NnShellKit, SwiftPickerKit, ConsoleProgressKit, and nn-cli-architecture. A skill that fails to install is listed in the report and does not fail the command.

If the command fails, report its error and stop. Do not create or repair these files by hand.

## 4. First command

Follow `Workflows/add-command.md` completely for the first command, with `--parent <tool>`. Run it from the same folder; it registers the command in the root's `configuration`.

## 5. Tests (only when requested)

`init` already created `MockContextFactory.swift` and `<Root>TestRunner.swift`, and the first command's workflow left its test files with only a `makeSUT`. When the request asks for tests, write them there, starting with one `testRun` test that runs the first command with `--help` or its happy path through `<Root>TestRunner` and `MockContextFactory`. Follow `TestingReference.md` in the NnArgumentParser skill.

## 6. Build and report

Run `swift build` in `<tool>/`. Fix errors in the files the first command's workflow wrote. Report the two package paths, the file list, and the command to run the tool.
