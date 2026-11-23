# nnscaffold

Deterministic code generation for swift projects

## Install

```sh
brew install nikolainobadi/nn/nnscaffold
```

## Generated CLI tests

`nnscaffold cli add-command`, `nnscaffold cli add-prompter`, and `nnscaffold cli add-use-case` use testing modules in generated tests when matching modules are dependencies of the tool's test target. Imports without a matching testing dependency stay unchanged.

New tools include an architecture check for tests importing production modules that have testing equivalents. Generated checks also cover command formatting, immediate picker confirmations, `UseCase` suffixes, dependency minimums, and warnings-as-errors settings.

Before extending an existing tool, run `nnscaffold cli status` from the repository root to compare its `.nn-cli-architecture` marker with nnscaffold's version without changing files. Use the `nn-cli-architecture` catch-up workflow to bring an older tool up to date. Generated commands use `NnFormattedWorkflowCommand` and `NnStatelessResultsFormatter`; existing tools need NnArgumentParser 2.1.0 or later before adding them.

`cli add-command`, `cli add-group`, `cli add-helper`, `cli add-prompter`, `cli add-use-case`, `cli add-flow-tests`, and `cli add-architecture-tests` require confirmation when the marker is behind, ahead, missing, or invalid. Without a terminal they stop; `--yes` does not bypass this check. Use `--ignore-architecture` to explicitly proceed anyway. `--preview` skips the check.

To update an existing tool's architecture checks, run this from the repository root:

```sh
nnscaffold cli add-architecture-tests
```

This replaces the tool's generated architecture checks. Use `--preview` to review the planned changes first. Replace flagged production imports with their testing modules and add any missing testing modules to the test target's dependencies.

## Releases

Builds and release notes for every version are on the [releases page](https://github.com/nikolainobadi/nnscaffold-releases/releases).
