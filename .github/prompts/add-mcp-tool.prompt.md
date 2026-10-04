---
description: "Scaffold a new tool function inside an existing MCP server."
agent: "agent"
argument-hint: "<tool_name>"
---
Add a new MCP tool named `{{tool_name}}` to the MCP server in this workspace:

- Create the tool function with a clear docstring and type-hinted parameters.
- Register it with the server so it is exposed over MCP.
- Add a minimal test that calls the tool directly and checks the output shape.
- Keep the change limited to this one tool — do not refactor unrelated code.
