---
name: nn-mcp-architecture
description: Architecture reference for Swift MCP servers built on MCPServerKit. Covers the entry point and startup rules, the feature and capability layout, the protocol and backend behind each tool handler, where tool errors are translated, per-tool annotations, handler test shape, and registering a server with a client. USE WHEN creating a new Swift MCP server, adding a feature, tool, or capability to one, planning any of those, deciding where a type or file belongs in an MCP server, or reviewing an MCP server's structure. Not for the MCPServerKit API itself (its own skill owns that), MCP servers in other languages, or Swift command-line tools built on NnArgumentParser.
---

# nn-mcp-architecture

Rules for how a Swift MCP server on MCPServerKit is structured. Each rule is written once, here; the smells that break them are in the Don't list.

Paths in this file resolve from the directory containing this `SKILL.md`. Paths in the server being built are always written as project paths.

## Scope and the sibling skill

This skill governs structure: which types exist, what each may do, and where its file lives. It does not restate API surfaces. Before writing any descriptor, handler, or schema, read `ApiReference.md` in the **MCPServerKit** skill. If that skill is not loaded, install it with `claude plugin install MCPServerKit@nn-swift-skills`, then continue. Do not reproduce its content from memory.

## Rules

1. **`Main/<Name>.swift` is the composition root.** It holds `@main` and `makeFeatures()`, which builds every store and client and passes each feature its backend. `makeFeatures()` takes no parameters, because `nnscaffold mcp add-feature` finds it by that signature.
2. **Required configuration resolves before `server.start()`.** Resolve it inside `makeFeatures()` or a helper it calls. A missing value writes one line to stderr and exits 1, naming what to set.
3. **Nothing writes to stdout.** Stdout is the protocol stream, so there is no `print` under `Sources/`.
4. **A feature is one domain and the unit of deletion. A capability is one read or write surface, or one workflow, with exactly one descriptor enum and one handler.**

   ```
   Sources/<Name>/
     Main/<Name>.swift
     Features/<Feature>/
       Feature/<Feature>Feature.swift
       Shared/<Feature>Backend.swift
       Shared/<Feature>ToolError.swift
       Capabilities/<Capability>/
         <Capability>ToolDescriptor.swift
         <Capability>ToolHandler.swift
         <Feature>Backend+<Protocol>.swift
   Tests/<Name>Tests/
     Shared/ToolResultText.swift
     Features/<Feature>/<Capability>ToolHandlerTests.swift
   ```

   A handler test builds the handler in `makeSUT` with a mock conformer of its protocol and reads results through the shared `extractText`.
5. **A handler decodes arguments, calls its protocol, and encodes the result.** It holds no logic and names no concrete type. The protocol is declared at the bottom of the handler file under `// MARK: - Dependencies`.
6. **Each feature has one backend, and it stays `Sendable`.** Thin servers put store or client code directly in it; logic gets stores injected into it. Anything it holds must be `Sendable` too, so a store that isn't has to be made safe, such as by becoming an actor. Declare `Sendable` on the struct itself: Swift rejects gaining it from a conformance in another file. Never add a type that only forwards calls.
7. **Each feature has one `<Feature>ToolError`, and translation happens in the backend, never in the handler.** `CancellationError` and `MCPError` are rethrown before any catch-all, because the SDK handles them. Every message says what to do next.

   ```swift
   do {
       return try await store.readRecipes()
   } catch let error as CancellationError {
       throw error
   } catch let error as MCPError {
       throw error
   } catch {
       throw RecipesToolError.operationFailed.withDetails(["reason": error.localizedDescription], message: "Couldn't read recipes. Check that the store is reachable, then try again.")
   }
   ```
8. **Annotations are decided per tool, and `.init()` is never used.** Reads are read-only and idempotent. Adding is not destructive. Deleting is destructive and idempotent. Overwriting data the caller can't recover is destructive.
9. **The schema lists exactly what the handler reads,** because the kit doesn't validate against it. Tool names are snake_case, start with a verb, and are unique across the server, because the kit crashes at startup on a duplicate. A description says when to use the tool and what it returns.

## Routing

| Observable condition | Read, completely, before… |
| --- | --- |
| No `Package.swift` with an executable target depending on MCPServerKit, or the request is to start a server | [Workflows/new-mcp-server.md](Workflows/new-mcp-server.md) before creating any file |
| A server exists and the request needs a new feature | the **Add a feature** section of [Workflows/new-mcp-server.md](Workflows/new-mcp-server.md) before changing any file |
| A server exists and the request is to register it with a client | the **Register the server** section of [Workflows/new-mcp-server.md](Workflows/new-mcp-server.md) before running any command |
| The feature exists and the request adds a read or write tool or capability | [Workflows/add-capability.md](Workflows/add-capability.md) before planning or creating any file |
| The capability is backed by an MCP resource | `ResourceToolPattern.md` in the MCPServerKit skill; this capability is written by hand for now |
| A type is about to be named, or work is about to be split across types | the `nn-architecture` skill and [references/glossary.md](references/glossary.md) before naming or splitting |
| A "where does this go" question, or a request to review a server's structure | answer from this file: cite the rule and the smell, and report only the Don't list items that fail, by number |

If a routed file cannot be read, say which one and what it governs, and stop the step that depended on it. Continue the steps that do not.

## Don't list

Each item is the smell of the rule with the same number.

Each item is marked with how it's enforced. **Generated:** `nnscaffold` produces the correct form, so check only code changed after generation. **Checked:** a test fails when it's broken, so review skips it if the project has that test. **Judgment:** only reading the code can tell.

1. A store or client built outside `makeFeatures()` [Judgment], a `makeFeatures` that takes parameters, or a `@main` type that lives anywhere but `Main/` [Generated].
2. Required configuration read after `server.start()`, or a missing value that fails on the first tool call instead of at launch. [Judgment]
3. A `print` or other stdout write under `Sources/`. [Judgment]
4. A descriptor enum with more than one handler, a handler serving more than one descriptor enum [Generated], or a capability whose folder can't be deleted without breaking another feature [Judgment].
5. A handler with logic beyond decoding, calling, and encoding, or one that names a concrete store, client, or backend. [Judgment]
6. A second backend in a feature, a type that only forwards calls to another, or a backend conformance marked `@unchecked Sendable` to silence the compiler. [Judgment]
7. A `do`/`catch` in a handler, a catch-all that swallows `CancellationError` or `MCPError`, a second error type in a feature, or a message that doesn't say what to do next. [Judgment]
8. A tool using `.init()` annotations, or a hint that disagrees with what the tool does. [Judgment]
9. A schema that lists an argument the handler doesn't read or omits one it does, a tool name that isn't verb-first snake_case, or a description that doesn't say when to use the tool. [Judgment]

## Completion

A new server is done when the package builds, `Main/` holds the entry point, the first capability follows its workflow, and it is registered with the client the request names. An added feature or capability is done when it is wired into `makeFeatures()` or its feature's `capabilities` list, no `nnscaffold:` placeholder remains in its files, each tool's annotations have been decided, the package builds, and the Don't list passes. Write tests only when the request includes them. Report the files created or changed.
