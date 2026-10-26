# Workflow: Review a CLI

Use when the request is to review, audit, or validate an existing tool, a feature folder, or a file against this architecture. This workflow is review only: it reports what breaks a rule and how to fix it, in words.

## 1. Settle the scope

Decide what is under review: the whole tool, one feature folder, one group, or one file. Take it from the request. If it is not clear, ask before reading any code. Tests are in scope only when the request names them.

## 2. Read the rules

Read `references/architecture.md` completely, and `references/style.md` completely. The universal rules, stage rules, and Don't list are already in `SKILL.md`.

## 3. Map the code onto the stages

Read the code in scope. For each type, decide its role (root, factory, command, options, input, inspector, inspection interpreter, detector, collector, prompter, collection interpreter, resolver, use case, formatter, payload model, error) by what it does, not by its name or folder. Note every type whose behavior spans two roles, and every type whose name or folder disagrees with its role. Those mismatches are findings in their own right.

## 4. Check

1. Walk the Don't list in `SKILL.md` against every type in scope, using the roles from step 3.
2. Check each feature folder against `references/architecture.md` section 7, and each payload type against section 4.
3. Check each Swift file against `references/style.md`: header, comments, type body, MARK order, nesting, expressions, naming.
4. Tests only when in scope: check their layout and kind against `references/architecture.md` section 8. For test-file internals, defer to the NnArgumentParser skill's `TestingReference.md`.

## 5. Report

Report only violations. Do not list what passes, summarize the architecture, or praise.

One entry per violation:

- **Where**: file path and type or member.
- **Rule**: the Don't list number, the `SKILL.md` rule, or the reference file and section.
- **What**: one sentence naming the violation by role.
- **Fix**: one or two sentences describing the change in words.

Order entries by stage (root and dependencies, command, inspect, inspection interpreter, collect, collection interpreter, resolve, then errors, layout, style, interruption, concurrency, and progress, tests), and by file within a stage. Group repeats of one violation (a missing header in twelve files) into a single entry with the file list. If nothing in scope fails, say so in one sentence.

## Constraints

- Do not edit any file.
- Do not write Swift code, including "what it should look like" snippets.
- Do not restate a rule; cite it.
- Do not invent context the code does not show. When a check cannot be decided from the code (whether a second feature will use a file), say it is undecidable and why.
