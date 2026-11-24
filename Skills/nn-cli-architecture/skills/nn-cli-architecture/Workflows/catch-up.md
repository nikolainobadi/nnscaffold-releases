# Workflow: Catch Up a CLI

Use when the request is to bring an existing tool up to date with changes to this architecture or to what `nnscaffold` generates, or to list what in a tool needs refactoring after those changes. This workflow reports only: it produces a refactor list scoped to the changes the tool has not applied, and edits nothing but the marker file and the tool's exceptions file, each only when the user says to.

A general review against the whole architecture, with no mention of recent changes, belongs to [review-cli.md](review-cli.md) instead.

## 1. Find the tool, its marker, and the pending entries

The tool is the package folder whose `Package.swift` has an executable target depending on NnArgumentParser. If the working directory holds more than one, ask which one. The marker is `<tool>/.nn-cli-architecture`, a one-line file holding the version of this skill the tool was last brought up to, such as `0.7.1`.

The skill's own version is the `version` in `../../.claude-plugin/plugin.json`, resolved from the directory containing `SKILL.md`. Read `references/changelog.md` completely. Its version sections are newest first; compare versions by semantic version precedence, not as strings. Read `references/exceptions.md` completely, and the tool's exceptions file if it has one.

`nnscaffold cli status`, run from the folder holding the tool, makes the same marker comparison against the nnscaffold version and prints it, without the refactor list, including whether any pending change requires a catch-up. Use it when the user only asks whether the tool is behind.

An entry requires a catch-up when its **Requires catch-up** line says `yes`, or when it has no such line (entries released before the field existed).

- **No marker**: the tool predates the changelog. Follow [review-cli.md](review-cli.md) completely with the whole tool as its scope, then continue at step 5.
- **Marker is a version**: the pending entries are those under `## [Unreleased]` and under every version section newer than the marker. If there are none, or none of them requires a catch-up, and the marker is older than the skill's version, there is nothing to refactor: write the skill's version to the marker as step 5 describes, without waiting to be told, and say the tool is up to date with it. List each pending entry by title as an optional simplification, if there are any. If there are none and the marker is the skill's version, say the tool is up to date and stop. Otherwise continue at step 2.
- **Marker is not a version, or is newer than the skill's version**: say what the file holds, and ask whether to treat the tool as having no marker. Do not guess a version.

## 2. Read the current rules for each pending entry

For each pending entry, read the rule or section its **Rules** line names, in its current form in `SKILL.md` or the reference file. Check the current rule, not the entry's wording: a later entry may have changed it again. When a pending entry says it replaces an earlier pending entry, check only the later one.

When an entry's **Generated output** names an `nnscaffold` command, the code to compare against is what that command generates now. Describe the difference from the entry and the current rules; do not run `nnscaffold` against the tool to find out.

## 3. Find the affected code

A pending entry whose **Behind when** ends with "After `nnscaffold cli add-architecture-tests`, `swift test` lists each one." is checked by the tool's tests, so don't search for it. The same goes for a pending entry whose **Behind when** another pending entry's **Changed** says `ArchitectureRulesTests` now checks. Its finding is a single fix: run `nnscaffold cli add-architecture-tests`, then `swift test`, and fix each failure it lists. Don't run either command yourself.

For each other pending entry, search the tool for what its **Behind when** line describes. Read each match far enough to decide its role (step 3 of [review-cli.md](review-cli.md)) and whether it actually breaks the current rule. A search hit alone is not a finding. Match each finding against the tool's exceptions file, and check every entry in it for staleness, as `references/exceptions.md` describes.

Tests are in scope only when the request names them, as in a review.

## 4. Report

Group the report by pending entry, oldest first, under each entry's version and title. List its findings in the entry format of [review-cli.md](review-cli.md) step 5 (**Where**, **Rule**, **What**, **Fix**), with its grouping of repeats. An entry with no affected code gets one line saying so. Entries the tool's tests check, as step 3 describes, are listed together in one group with that single fix. An entry whose **Behind when** cannot be decided from the code says what is undecidable and why.

Entries that don't require a catch-up come after the others, in one group headed **Optional**, one line each naming the entry and what the tool may simplify.

After the pending entries, add the **Accepted exceptions** and **Stale exceptions** sections as [review-cli.md](review-cli.md) step 5 describes.

End with one line naming the version the marker will hold once the list is done: `Marker: <old or none> → <skill version>`.

## 5. Update the marker

Write the skill's version to `<tool>/.nn-cli-architecture` only when the user says the refactor is done, or says to record it as done without refactoring. Write the version alone, followed by a newline. Never update the marker in the same turn as the report unless the user asked for that up front.

## Constraints

- Do not edit any file except the marker, and that only under step 5 or when step 1 finds nothing pending, and the tool's exceptions file, and that only when the user approves an entry (**Adding an entry** in `references/exceptions.md`).
- Do not write Swift code, including "what it should look like" snippets.
- Do not report violations unrelated to a pending entry. A tool with no marker is the exception: its catch-up is a full review.
- Do not invent changelog history. If the user describes a change the changelog does not have, say the entry is missing and that it belongs under `## [Unreleased]` in `references/changelog.md`.
