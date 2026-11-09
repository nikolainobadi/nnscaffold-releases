# nn-cli-architecture

A Claude Code skill that teaches how a Swift command-line tool built on [NnArgumentParser](https://github.com/nikolainobadi/NnArgumentParser) is structured: one root command with a dependency factory, every command as an `NnWorkflow` pipeline (inspect, interpret, collect, interpret, perform), a feature-folder layout, and a file style. It states rules and the smell that breaks each one. It names no particular tool.

## Contents

- [What it covers](#what-it-covers)
- [Installation](#installation)
- [Invoking it](#invoking-it)
- [Files](#files)
- [Dependencies](#dependencies)
- [Maintenance](#maintenance)

## What it covers

- Where every stage type lives and what each stage may and may not do.
- The folder layout for `Main/` and `Features/<Feature>/`.
- File style: header, MARK order, nesting, naming, no default parameters, no inline comments.
- Four procedures: starting a new CLI, adding a command to an existing one, reviewing an existing tool, and catching a tool up after the architecture changes.
- A numbered review checklist (the Don't list), one item per rule's smell.
- A flow file per command (`Flows/<command path>.md`): a short plain-English account of what the command does, what it asks, and when it stops, with a generated test that keeps one file per command.
- A changelog of rule and generated-output changes, and a marker file in each tool recording the last change it applied.

It does not restate the NnArgumentParser API. The skill routes to the NnArgumentParser skill for signatures.

## Installation

The skill ships as a plugin in the `nn-swift-skills` marketplace. Register the marketplace, then install it into a project:

```bash
claude plugin marketplace add nikolainobadi/nn-swift-skills
claude plugin install nn-cli-architecture@nn-swift-skills --scope project
```

Installing it declares a dependency on `NnArgumentParser@nn-swift-skills`, which comes from the same marketplace.

## Invoking it

The skill activates on its own for requests about Swift CLI structure. Examples:

- "Create a new Swift CLI called `relnotes` with a `generate` command" → follows `Workflows/new-cli-project.md`, then `Workflows/add-command.md`.
- "Add a `prune` command to this tool" → produces the stage-plan table, the file table, then the files.
- "Where should the version detector live?" → answers from `references/architecture.md` and cites the rule.
- "Review this command against the architecture" → follows `Workflows/review-cli.md`: maps each type to its stage, walks the Don't list, the folder layout, and the file style, and reports only violations (where, rule, what, fix). It edits nothing.
- "Catch this CLI up to the latest nn-cli-architecture" or "What needs refactoring after the recent nnscaffold changes?" → follows `Workflows/catch-up.md`: reads the tool's `.nn-cli-architecture` marker, takes the changelog entries newer than the version it holds, and reports the code each one affects, grouped by entry. It updates the marker only when you say the refactor is done.

- "Add flow files to this CLI" → follows **Adding flow files to an existing tool** in `references/flow-format.md`: runs `nnscaffold cli add-flow-tests`, then writes a flow file for each command `swift test` lists.

Expected results are files in the tool's feature folder, a stage-plan table in the reply, and a build. Tests are added only when asked for.

## Files

| File | Role |
| --- | --- |
| `skills/nn-cli-architecture/SKILL.md` | Entry file: scope, universal rules, stage rules, routing, Don't list, completion |
| `skills/nn-cli-architecture/references/architecture.md` | Entry point, dependencies, command shape, stage payload placement, module boundaries, errors, folder layout, testing shape |
| `skills/nn-cli-architecture/references/style.md` | File style and naming |
| `skills/nn-cli-architecture/Workflows/new-cli-project.md` | Procedure for a new tool |
| `skills/nn-cli-architecture/Workflows/add-command.md` | Procedure for a new or reshaped command, with an illustrative file table |
| `skills/nn-cli-architecture/Workflows/review-cli.md` | Review-only procedure for auditing a tool, feature, or file, with the report format |
| `skills/nn-cli-architecture/Workflows/catch-up.md` | Refactor list for a tool, scoped to the changelog entries it has not applied |
| `skills/nn-cli-architecture/references/flow-format.md` | Flow file location, template, rules, how to trace a command into one, an example, and the steps for adding flow files to an existing tool |
| `skills/nn-cli-architecture/references/changelog.md` | One entry per change to the rules or to `nnscaffold`'s generated output, grouped by release version |
| `.claude-plugin/plugin.json` | Plugin manifest |

## Dependencies

- The NnArgumentParser skill (`NnArgumentParser@nn-swift-skills`) for API reference.
- The SwiftPickerKit, NnShellKit, and NnFileKit skills for the picker, shell, and file-system APIs, when a tool uses them.

## Maintenance

The source lives at `nnscaffold/Skills/nn-cli-architecture/` in the CodeScaffold repository. It is published with each nnscaffold release: `nnex ship tool` sets `version` in `plugin.json` to the release version, copies the skill into the public releases repo, and pins the `nn-swift-skills` marketplace entry to the release tag. When the NnArgumentParser workflow API changes, update the stage rules here in the same pass as that skill's `WorkflowReference.md`.

Every change to the rules or to what `nnscaffold` generates adds an entry under `## [Unreleased]` in `references/changelog.md` in the same change. Before each nnscaffold release, move those entries into a new `## [<version>] - <date>` section directly below `## [Unreleased]`, by hand. Each tool's `.nn-cli-architecture` file holds the skill version it was last brought up to: `Workflows/new-cli-project.md` writes it for a new tool, and `Workflows/catch-up.md` updates it after a catch-up.
