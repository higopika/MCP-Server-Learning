---
name: mcp-server-scaffolding
description: 'Use when the user wants to scaffold, create, or bootstrap a new custom MCP (Model Context Protocol) server from scratch, including the manifest, entrypoint, and an example tool.'
---

# MCP Server Scaffolding

## When to Use
- Starting a brand-new MCP server project.
- Adding the standard folder layout (entrypoint, manifest, tools, tests) to an existing project that doesn't have one yet.

## Procedure
1. Create the project folder with a `pyproject.toml` (or `package.json` for Node) declaring the MCP SDK dependency.
2. Create the server entrypoint (e.g. `server.py`) that registers at least one example tool.
3. Define the tool using the SDK's decorator/schema pattern — include a docstring and typed parameters so the schema is generated correctly.
4. Add a `README.md` with run instructions.
5. Add a minimal test that invokes the tool function directly (without going through the protocol transport).

## Reference
- See [MCP server template](./assets/server_template.py) for the base entrypoint shape (create this file when first used).
