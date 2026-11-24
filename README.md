# nnscaffold

Deterministic code generation for swift projects

## Install

```sh
brew install nikolainobadi/nn/nnscaffold
```

## Generated CLI tests

`nnscaffold cli add-command`, `nnscaffold cli add-prompter`, and `nnscaffold cli add-use-case` use testing modules in generated tests when matching modules are dependencies of the tool's test target. Imports without a matching testing dependency stay unchanged.

New tools include an architecture check for tests importing production modules that have testing equivalents. Generated checks also cover command formatting, immediate picker confirmations, `UseCase` suffixes, dependency minimums, and warnings-as-errors settings.

Before extending an existing tool, run `nnscaffold cli status` from the repository root to compare its `.nn-cli-architecture` marker with nnscaffold's version without changing files. Use the `nn-cli-architecture` catch-up workflow to bring an older tool up to date. Generated commands use `NnWorkflowCommand` and `NnStatelessResultsFormatter`; `cli init` requires NnArgumentParser 3.0.0 or later. Before adding generated commands to an existing tool, rename its commands' conformance from `NnFormattedWorkflowCommand` to `NnWorkflowCommand` and raise its NnArgumentParser requirement to 3.0.0 together.

`cli init` generates a ConsoleProgressKit dependency of 0.9.0 or later. For terminal-width formatting in an existing tool, raise its ConsoleProgressKit minimum to 0.9.0. The `nn-cli-architecture` workflow allows results formatters to hold `outputWidth: Int?` alongside their `TextStyler`, supplied by the command's computed `resultsFormatter` using `TerminalCapabilities.size(for: .standardOutput)?.width`. Read terminal facts directly from `TerminalCapabilities`, rather than the context factory. When the width is `nil`, leave output unwrapped and untruncated.

`cli add-command`, `cli add-group`, `cli add-helper`, `cli add-prompter`, `cli add-use-case`, and `cli add-flow-tests` require confirmation when an older marker indicates a required architecture catch-up, or when the marker is ahead, missing, or invalid. `cli status` distinguishes an older marker with no catch-up required; when no pending architecture change requires one, the catch-up workflow records the new version without a refactor report. When confirmation is required, they stop without a terminal; `--yes` does not bypass this check. Use `--ignore-architecture` to explicitly proceed anyway. `--preview` skips the check.

When adopting NnArgumentParser 3.0.0, replace uses of `NnCheck`, `NnRunnableCheck`, `NnCheckGroup`, and `NnCheckedWorkflow` with shared inspector helpers and interpreter rules used in each command's inspection. Group assemblies no longer build stages. A shared precondition that prompts needs a type local to the tool and a recorded architecture exception.

To update an existing tool's architecture checks, run this from the repository root:

```sh
nnscaffold cli add-architecture-tests
```

This replaces the tool's generated architecture checks regardless of its architecture marker, without asking for confirmation, including when no terminal is available. This command no longer accepts `--ignore-architecture`; remove that flag from scripts and invocations. Use `--preview` to review the planned changes first. Generated checks require NnArgumentParser 3.0.0 or later and flag commands conforming to `NnFormattedWorkflowCommand` or `NnCustomResultsWorkflowCommand`, as well as commands that implement their own `handleResults`. Replace flagged production imports with their testing modules and add any missing testing modules to the test target's dependencies.

## Releases

Builds and release notes for every version are on the [releases page](https://github.com/nikolainobadi/nnscaffold-releases/releases).