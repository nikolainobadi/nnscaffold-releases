# nn-mcp-architecture

A Claude Code skill that teaches how a Swift MCP server built on [MCPServerKit](https://github.com/nikolainobadi/MCPServerKit) is structured: one composition root, features as the unit of deletion, a capability per read or write surface, and a protocol and backend behind every tool handler. It states rules and the smell that breaks each one.

## What it covers

- The entry point and what must resolve before the server starts.
- The folder layout for features, capabilities, and handler tests.
- What a handler may do, and where tool errors are translated.
- How to decide annotations, schemas, tool names, and descriptions.
- A numbered review checklist (the Don't list), one item per rule.

It does not restate the MCPServerKit API. The skill routes to the MCPServerKit skill for signatures.

## Installation

The skill ships as a plugin in the `nn-swift-skills` marketplace. Register the marketplace, then install it into a project:

```bash
claude plugin marketplace add nikolainobadi/nn-swift-skills
claude plugin install nn-mcp-architecture@nn-swift-skills --scope project
```

## Invoking it

The skill activates on its own for requests about Swift MCP server structure. Examples:

- "Create a Swift MCP server called `TodoServer` with a tool that reads recipes" → settles the name and tool surface, then scaffolds and fills in the server.
- "Where should the code that talks to the recipe store live?" → answers from the rules and cites the rule.
- "Review this server's structure" → reports the Don't list items that fail, by number. It edits nothing.

Expected results are files in the server's feature folders, and a package that builds. Tests are added only when asked for.

## Files

| File | Role |
| --- | --- |
| `skills/nn-mcp-architecture/SKILL.md` | Entry file: scope, rules, routing, Don't list, completion |
| `skills/nn-mcp-architecture/Workflows/new-mcp-server.md` | Creating a server, adding a feature, and registering a server with a client |
| `skills/nn-mcp-architecture/Workflows/add-capability.md` | Adding a read or write capability to a feature and filling in its stubs |
| `.claude-plugin/plugin.json` | Plugin manifest |

## Dependencies

The MCPServerKit skill (`MCPServerKit@nn-swift-skills`) for API reference.

## Limits

- It covers read and write capabilities. A resource-backed capability is written by hand from the MCPServerKit skill's `ResourceToolPattern.md`, and prompts aren't covered.
- It has no workflow for reviewing a server in depth or for moving an existing server onto these rules.
- It applies only to Swift servers on MCPServerKit.

## Maintenance

The source lives at `nnscaffold/Skills/nn-mcp-architecture/` in the CodeScaffold repository. Don't change the version in `plugin.json` by hand.
