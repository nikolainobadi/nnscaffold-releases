# Workflow: Add a Capability

Use when a server and its feature exist and the request adds a read or write tool. Produces one capability: a descriptor, a handler with its protocol, the backend's conformance, and a handler test file, wired into the feature. The files come from `nnscaffold`; only the stubs inside them are filled in by hand.

Read `ApiReference.md` in the MCPServerKit skill before step 4. If that skill isn't loaded, say so and stop at step 4.

## 1. Settle the tool surface

From the request, fix before running anything:

- the feature it belongs to. If none fits, follow **Add a feature** in [new-mcp-server.md](new-mcp-server.md) first.
- the kind: `read` when every tool only reads, `write` when every tool changes state. A surface that does both is two capabilities (rule 4).
- the noun: PascalCase, such as `Recipe`. With the kind it names the capability: `Recipe` and `write` give `RecipeWrite/`, `RecipeWriteToolHandler`, and `RecipeWriter`.
- each tool: a verb-first snake_case name unique across the server, the arguments it reads, and what it returns (rule 9).

Ask only for what the request leaves open.

## 2. Create the capability

Run from the package folder, or from the folder directly above it, with one `--tool` per tool:

```
nnscaffold mcp add-capability <Noun> --feature <Feature> --kind <read|write> --tool <tool_name> [--tool <tool_name>] --non-interactive [--server <folder>] [--header]
```

Pass `--server` and `--header` under the same conditions as in [new-mcp-server.md](new-mcp-server.md).

It creates, under `Sources/<Name>/Features/<Feature>/Capabilities/<Noun><Kind>/`:

- `<Noun><Kind>ToolDescriptor.swift`, with one case per tool.
- `<Noun><Kind>ToolHandler.swift`, with one private method per tool and the protocol under `// MARK: - Dependencies` (rule 5).
- `<Feature>Backend+<Protocol>.swift`, the backend's conformance with one stub per tool (rule 6).

It also creates `Tests/<Name>Tests/Features/<Feature>/<Noun><Kind>ToolHandlerTests.swift`, with a `makeSUT` and a mock conformer and no test cases, and adds `.init(handler: <Noun><Kind>ToolHandler(<reader|writer>: backend))` to the feature's `capabilities` list.

The command stops, writing nothing, when the feature doesn't exist, when its Feature file has no `module(using backend:)`, when the capability or a tool name already exists, or when a tool name isn't snake_case. Report its error and stop this step. Do not write the files by hand to get past it.

## 3. Find the stubs

Run the search the report prints. Every line it lists is a stub to replace, and the capability isn't done while any remain.

## 4. Replace the stubs, in this order

For each tool:

1. **Description** in the descriptor: when to use the tool and what it returns (rule 9).
2. **Schema** in the descriptor: replace `Schema.object([:])` with exactly the arguments the handler will read (rule 9).
3. **Protocol method** in the handler file: give it the arguments and a return type. Keep it `async throws` unless the backend can't throw. Change the matching method of the `Mock<Protocol>` in the generated test file to the same signature, or the test target stops compiling.
4. **Handler method**: decode the arguments, call the protocol, and encode the result. Nothing else (rule 5).
5. **Backend method** in the conformance file: replace the `fatalError` with the store or client call, translating errors into the feature's `<Feature>ToolError` with the pattern in rule 7.
6. **Annotations** in the descriptor: decide each tool's hints (rule 8). A write capability starts every tool as destructive and not idempotent; change each hint that doesn't match what the tool does. Adding is not destructive. Deleting is destructive and idempotent.

If the backend needs a store or client, build it in `makeFeatures()` and pass it to the backend there (rules 1 and 6).

## 5. Tests

Write tests only when the request includes them. Put them in the generated test file, build the handler with `makeSUT`, give the mock what each case needs, and read results through the shared `extractText`.

## 6. Build and check

Run `swift build --build-tests` in the package, so the test target's mock is compiled too, then `swift test` if tests were written. Fix what fails, run the stub search again to confirm it finds nothing, and check the capability against the Don't list in `SKILL.md`.
