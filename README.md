# nnscaffold

Deterministic code generation for swift projects

## Install

```sh
brew install nikolainobadi/nn/nnscaffold
```

## Generated CLI tests

`nnscaffold cli add-command`, `nnscaffold cli add-prompter`, and `nnscaffold cli add-use-case` use testing modules in generated tests when matching modules are dependencies of the tool's test target. Imports without a matching testing dependency stay unchanged.

New tools include an architecture check for tests importing production modules that have testing equivalents. To update an existing tool's architecture checks, run this from the repository root:

```sh
nnscaffold cli add-architecture-tests
```

This replaces the tool's generated architecture checks. Use `--preview` to review the planned changes first. Replace flagged production imports with their testing modules and add any missing testing modules to the test target's dependencies.

## Releases

Builds and release notes for every version are on the [releases page](https://github.com/nikolainobadi/nnscaffold-releases/releases).