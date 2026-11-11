# Architecture Exceptions

A tool can record the exceptions to this architecture that the user has approved, so a review or catch-up knows a departure was deliberate instead of reporting it again. How to ask for an exception is in the `nn-architecture` skill, Exceptions. This file says where an approved one is recorded and how the workflows read it.

## Location

`<tool>/.nn-cli-architecture-exceptions.md`, beside the catch-up marker `<tool>/.nn-cli-architecture`. A tool without the file has no exceptions.

## Entry format

One section per exception:

```
## <short title>

- **Rule**: the rule it departs from, cited the way a review finding cites it (a Don't list number, a `SKILL.md` rule, or a reference file and section).
- **Applies to**: the type names it covers, each with a member when only that member departs. Never line numbers.
- **Reason**: why the rule doesn't fit this tool.
- **Tried**: what was tried within the rule, and why it was worse.
- **Approved at**: the version of this skill when the user approved it, found as step 1 of [catch-up.md](../Workflows/catch-up.md) describes.
```

## Matching

A finding is covered when its rule is the entry's **Rule** and its type is listed under **Applies to** (and its member, when the entry names one). A covered finding is reported under **Accepted exceptions**, not as a violation.

## Stale entries

An entry is stale when either holds:

- a type under **Applies to** no longer exists in the tool
- a version section of the changelog newer than **Approved at** names the entry's rule under **Rules**

A stale entry covers nothing. Its findings are reported as violations, and the entry is reported under **Stale exceptions** with the type that's gone or the changelog entry that changed the rule.

## Adding an entry

Only the user approves an entry. When a finding looks deliberate, propose an entry in the format above, after the report. Write it to the file only when the user says to.

When the rule is already excepted in another tool the user names, or the user says the departure would be the same anywhere, the rule itself should change: propose an entry under `## [Unreleased]` in [changelog.md](changelog.md) instead of another exception.
