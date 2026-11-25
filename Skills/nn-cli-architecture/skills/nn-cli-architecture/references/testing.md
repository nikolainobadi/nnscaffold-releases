# Testing

Contents: 1. What the test plan lists · 2. Writing the tests · 3. Shared test helpers

Every command is planned and built with its tests. This file is the one place that says which tests a command gets and how they are written; other files link here instead of restating it. Where test files live and which tests `nnscaffold` generates are in section 8 of `references/architecture.md`. Read `TestingReference.md` in the NnArgumentParser skill for the API before writing any test.

## 1. What the test plan lists

The plan shown in step 2a of `Workflows/add-command.md` includes a test plan: for each test file the `--preview` list creates, the tests it will hold, one line each, named for the behavior it checks. Derive them from the spec and the behavior notes:

| From | Tests |
| --- | --- |
| Each `errors` case | an interpreter test that the blocking fact throws it |
| Each requirement | an inspection interpreter test for `.known` from each flag or fact that settles it, and one for `.unknown` with the Info it carries |
| Each findings field the inspection interpreter derives rather than copies | an inspection interpreter test of the derivation |
| The Brief | a collection interpreter test that answers and findings build it, and one for each field set only on some branches |
| Each conditional skip in the collector | a collector test for each branch |
| Each `results` case | a formatter test of its message, and a performer test of the step sequence that ends in it |
| Each use case method | a test for each behavior its behavior notes describe, including each failure it reports |
| Each helper | its spec `examples`, which are generated; list only cases the examples miss |
| Each `validate()` conflict | a command test that the pair is rejected |
| The command | one `testRun` test of the happy path through the root |

A change to an existing command (a flag, a use case, a reshape) plans the tests for the rows it touches. A file in the `--preview` list that gets no tests says so in the plan, with the reason.

## 2. Writing the tests

Write the planned tests in step 5 of `Workflows/add-command.md`, into the files the report lists under `Tests to write:`, using the generated `makeSUT` and test factories. Test code may use default parameter values; production code may not.

Four kinds of test, in order of preference:

1. **Interpreters as pure functions.** Build the facts (or findings, requirements, answers) by hand, call `interpret`, assert on the result or the thrown `<Feature>Error`. No mocks needed.
2. **Workflow wiring.** Build a `<Feature>Workflow` with `MockInspector`, `MockCollector`, and `MockPerformer` around the real interpreters, run it, and assert on `capturedRequirements` and `capturedBriefs`.
3. **Whole command.** Drive the root with `try await testRun`, passing the test factory vending mock infrastructure, the command's arguments, and the interaction mode explicitly; assert on the captured `NnOutput` output or on what the mocks recorded. Progress written through a display is not captured; the test factory vends `MockProgressDisplay` from the `ConsoleProgressTesting` product, and the test asserts on what it recorded (`trackedTitles`, each activity's `updates` and `outcome`, `notes`). A use-case test injects its own `MockProgressDisplay` the same way. A tracked inspector's title shows up in `trackedTitles` on the factory's display, and the inspector's own tests need no mock display. Interrupt cleanups are tested through `testRunCapturingCleanups`, which returns them without running them; run them and assert `terminalWasRestored` on the factory's display.
4. **Performers and use cases.** Test each use case for its own knowledge, through its public surface, with real collaborators and mocks only for what fronts something outside the program. Test a performer only for its sequence: the order of its steps, what each step receives, and that a failure stops the steps after it. The Testing section of the `nn-architecture` skill has the rule.

## 3. Shared test helpers

When commands share a `<Concern>Settings`, the test factory has two helpers for it: one that takes every field, for tests that check those fields, and short ones that fix the fields a test does not care about (`make<Concern>Settings(previewRequested:)`). Tests that do not check the shared values use the short helpers, so adding a field to the settings changes only the factory and the tests that check it.
