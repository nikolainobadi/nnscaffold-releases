# Workflow: Review a CLI

Use when the request is to review, audit, or validate an existing tool, a feature folder, or a file against this architecture. This workflow is review only: it reports what breaks a rule and how to fix it, in words.

## 1. Settle the scope

Decide what is under review: the whole tool, one feature folder, one group, or one file. Take it from the request. If it is not clear, ask before reading any code. Tests are in scope only when the request names them.

## 2. Read the rules

Read `references/architecture.md` completely, `references/style.md` completely, `references/flow-format.md` completely, and `references/exceptions.md` completely. Read the tool's exceptions file too, if it has one. The universal rules, stage rules, and Don't list are already in `SKILL.md`.

## 3. Map the code onto the stages

Read the code in scope. For each type, decide its role (root, factory, command, options, input, inspector, inspection interpreter, detector, collector, prompter, collection interpreter, performer, use case, formatter, payload model, error) by what it does, not by its name or folder. Note every type whose behavior spans two roles, and every type whose name or folder disagrees with its role. Those mismatches are findings in their own right.

## 4. Check

1. Walk the Don't list in `SKILL.md` against every type in scope, using the roles from step 3. Follow each item's mark: for Generated, check only code changed after generation; for Checked, skip the item if the project has that test (`ArchitectureRulesTests.swift` or `FlowFilesTests.swift`), and check it by reading if not; for Judgment, read the code closely.
2. Check each feature folder against `references/architecture.md` section 7, and each payload type against section 4.
3. Check each Swift file against `references/style.md`: header, comments, type body, MARK order, nesting, expressions, naming. The header check follows the project's convention: a file without a header is a violation only when the project's other Swift files have one.
4. Tests only when in scope: check their layout and kind against `references/architecture.md` section 8. For test-file internals, defer to the NnArgumentParser skill's `TestingReference.md`.
5. Flow files: each leaf command in scope has `Flows/<command path>.md`, and it matches the code. Compare its steps, questions, stops, and Code line against the files `references/flow-format.md` traces them from, and check it against that file's rules.
6. Exceptions: match each finding against the tool's exceptions file, and check every entry in it for staleness, as `references/exceptions.md` describes.

## 5. Report

Report only violations, plus the two exceptions sections below. Do not list what passes, summarize the architecture, or praise.

One entry per violation:

- **Where**: file path and type or member.
- **Rule**: the Don't list number, the `SKILL.md` rule, or the reference file and section.
- **What**: one sentence naming the violation by role.
- **Fix**: one or two sentences describing the change in words.

Order entries by stage (root and dependencies, command, inspect, inspection interpreter, collect, collection interpreter, perform, then errors, layout, style, interruption, concurrency, and progress, tests, flow files), and by file within a stage. Group repeats of one violation (a missing header in twelve files) into a single entry with the file list. If nothing in scope fails, say so in one sentence.

After the violations, list each finding an exceptions entry covers under **Accepted exceptions**, one line each naming the entry's title and reason, and each stale entry under **Stale exceptions**, one line each naming the type that's gone or the changelog entry that changed the rule. Leave out a section with nothing in it.

## Constraints

- Do not edit any file, except to write an entry to the tool's exceptions file after the report, when the user approves it (**Adding an entry** in `references/exceptions.md`).
- Do not write Swift code, including "what it should look like" snippets.
- Do not restate a rule; cite it.
- Do not invent context the code does not show. When a check cannot be decided from the code (whether a second feature will use a file), say it is undecidable and why.
