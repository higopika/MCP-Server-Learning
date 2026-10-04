## Agent Customization (`.github/`)

GitHub Copilot's agent can be customized with several file types under
`.github/`. Each one is a different primitive. Some load automatically,
others are invoked on demand.

| Primitive | File(s) in this repo | Loaded when |
|---|---|---|
| Agent instructions | [copilot-instructions.md](.github/copilot-instructions.md) | Always, every chat request in this workspace |
| File instructions | [instructions/python.instructions.md](.github/instructions/python.instructions.md) | Automatically, when a `*.py` file is being edited |
| Prompt | [prompts/add-mcp-tool.prompt.md](.github/prompts/add-mcp-tool.prompt.md) | On demand, via `/add-mcp-tool` in chat |
| Skill | [skills/mcp-server-scaffolding/SKILL.md](.github/skills/mcp-server-scaffolding/SKILL.md) | Automatically, when a request matches its description (for example "scaffold an MCP server") |
| Custom agent | [agents/mcp-debugger.agent.md](.github/agents/mcp-debugger.agent.md) | Invoked explicitly, or delegated to as a read-only sub-agent |
| Hook | [hooks/example-format.json](.github/hooks/example-format.json) | Deterministically, after every tool call (`PostToolUse`) |

Each file's frontmatter `description` is what lets the agent discover it
without being told the exact filename. Keyword-rich descriptions matter.

## MCP (Model Context Protocol)

MCP lets an AI agent call tools exposed by an external server (a database
query, an API call, a filesystem operation, and so on) through a standard
protocol, instead of each tool being hand-coded into the agent.

### Why Not Just Run the Script Directly

Mechanically, an MCP server over stdio is just a subprocess talking over
stdin/stdout, for example `python weather_server.py`. The protocol on top
is what matters: it standardizes how a tool describes itself (name,
description, typed inputs, generated straight from your function's type
hints and docstring) and how any client calls it (`initialize`, list
tools, call tool).

The USB-C analogy: electricity still just flows through wires either
way. The point of USB-C is that any compliant cable works with any
compliant device, without custom wiring per pair. MCP is the same idea
for "AI agent calls a function." The wire is still a subprocess and
stdin/stdout, but the connector shape (the protocol) is what let VS
Code call `get_forecast` correctly without ever reading
`weather_server.py`'s source, and would let any other MCP-compatible
host (Claude Desktop, Cursor, and so on) use the same file unmodified.

### Is an MCP Server Just a Tool

No, a tool is the individual callable capability (a function with a
name, a description, and a typed schema, for example `get_forecast`). An
MCP server is the process that hosts and exposes one or more tools (plus
two other primitives MCP supports but this repo has not used yet,
resources and prompts) over the protocol. A server is a container for
tools, not a tool itself.

| | What it is |
|---|---|
| `weather_server.py` | One MCP server |
| `get_alerts`, `get_forecast` | Two tools inside that one server |
| `fetch`, `git`, `time`, `sqlite` | Four separate MCP servers, each exposing one tool of the same name as the server |
| My built-in `read_file`, `run_in_terminal`, and so on | Tools too, just not MCP ones. They are built directly into the Copilot Chat extension instead of delivered over the protocol |

"Tool" is the generic concept, any callable capability an agent can
invoke. MCP is one standardized way of packaging and delivering tools so
they work across any compatible host, not the only way a tool can exist.

### How the Agent Picks the Right Tool

When a server has more than one tool, for example `weather_server.py`
has both `get_alerts` and `get_forecast`, the choice between them is
made by the AI model, not by the server or by fixed rules.

1. **Discovery**: on startup, the server replies to a `tools/list`
   request with every tool's name, description, and input schema,
   generated from the function signature and docstring.
2. **Context**: those descriptions, from every connected server plus
   built-in tools, are added to the model's context as a menu. The model
   never sees the Python source behind a tool, only that menu.
3. **Selection**: the model matches the user's request against the menu
   using natural language understanding, the same judgment call it uses
   to choose between any two of its own built-in tools. "Any weather
   alerts for Texas" matches `get_alerts`, because its description
   mentions alerts and its one parameter, a state code, fits "Texas."
   "Weather forecast for Seattle" matches `get_forecast` instead,
   because that request needs coordinates, not a state code.
4. **Routing**: once chosen, the host looks up which server owns that
   tool name and sends the call there.

The practical implication: **the description text is the entire
decision signal**. There is no keyword list or routing table. Vague,
overlapping descriptions increase the odds of the wrong tool being
picked, sharp descriptions and specific schemas (like `state: str` vs
`latitude: float, longitude: float`) reduce it.

### `.vscode/mcp.json` Is VS Code Specific, the Server Isn't

MCP standardizes the protocol a server speaks, not the file format a
host uses to register one. `.vscode/mcp.json` is VS Code's own config
file and schema. Every other MCP-capable host invented its own:

| Host | Where servers get registered | Key name used |
|---|---|---|
| VS Code | `.vscode/mcp.json` (workspace) or a user-level `mcp.json` | `servers` |
| Claude Desktop | `claude_desktop_config.json` in its own app data folder | `mcpServers` |
| Claude Code (CLI) | `.mcp.json` in the project, or `claude mcp add` | `mcpServers` |
| Cursor | `.cursor/mcp.json` | `mcpServers` |

Notice even the key name differs (`servers` vs `mcpServers`), so a
config file is not copy-pasteable between hosts as-is.

What *is* portable is the server itself. `weather_server.py` does not
know or care which host launched it, it just reads stdin and writes
stdout in the MCP protocol. The same file, completely unmodified, would
work from a `claude_desktop_config.json` entry too:

```json
{
  "mcpServers": {
    "weather": {
      "command": "C:\\path\\to\\.venv\\Scripts\\python.exe",
      "args": ["C:\\path\\to\\weather_server.py"]
    }
  }
}
```

The config file is a small, host-specific adapter. The server code is
the genuinely write-once, portable part.

### Existing vs Custom Servers

The distinction is about who wrote the server's code, not who wrote the
configuration pointing at it.

| | Who writes the code | Example in this repo |
|---|---|---|
| Existing server | Someone else, already published | `mcp-server-fetch` |
| Custom server | You | Not built yet, parked for a later session |

Either way, registering a server in `.vscode/mcp.json` is the same
mechanism. The only difference is whether `command`/`args` point at a
package you installed or a script you wrote yourself.

### Using an Existing MCP Server

This repo uses `mcp-server-fetch`, an official MCP reference server that
fetches a URL and converts it to Markdown. These are the exact steps that
were followed to set it up and verify it.

1. Set up a Python environment for the workspace (this created a
   `.venv` folder, excluded from git in `.gitignore`).
2. Install the server into that environment:

   ```powershell
   pip install mcp-server-fetch
   ```

3. Register it in [.vscode/mcp.json](.vscode/mcp.json):

   ```json
   {
     "servers": {
       "fetch": {
         "command": "${workspaceFolder}/.venv/Scripts/python.exe",
         "args": ["-m", "mcp_server_fetch"]
       }
     }
   }
   ```

   Using the full path to the virtual environment's `python.exe` with
   `-m mcp_server_fetch` avoids relying on the package's console script
   being on your system PATH.
4. Open `.vscode/mcp.json` in the editor and use the `Start` link shown
   above the `fetch` entry.
5. Confirm it is running: open the Chat view, click the tools icon
   (Configure Tools), and look for `fetch` listed alongside `Built-In`
   and any other MCP servers (for example `pylance mcp server`, added
   automatically by the Pylance extension, not by this repo). Clicking
   the MCP server picker shows each server's live status, `Running` next
   to `fetch` confirms it started successfully.
6. Ask the agent to fetch a web page (for example, "fetch
   https://example.com and summarize it").

This was verified end to end: `fetch` showed `Running` in the server
picker, and a real fetch request returned a live page summary with the
`fetch` tool call visible in the chat transcript.

### More Existing Server Examples

Three more official reference servers, installed and verified the same
way. Each is just another entry under `servers` in `.vscode/mcp.json`,
confirmed working in this repo.

**Git**, exposes repository tools such as status, diff, and log:

```powershell
pip install mcp-server-git
```

```json
"git": {
  "command": "${workspaceFolder}/.venv/Scripts/python.exe",
  "args": ["-m", "mcp_server_git", "--repository", "${workspaceFolder}"]
}
```

**Time**, timezone conversions and the current time:

```powershell
pip install mcp-server-time
```

```json
"time": {
  "command": "${workspaceFolder}/.venv/Scripts/python.exe",
  "args": ["-m", "mcp_server_time"]
}
```

**SQLite**, run queries against a local database file. Unlike the other
three, it ships only a console script, not a `-m`-runnable module, so the
command points straight at that script instead of `python.exe`:

```powershell
pip install mcp-server-sqlite
```

```json
"sqlite": {
  "command": "${workspaceFolder}/.venv/Scripts/mcp-server-sqlite.exe",
  "args": ["--db-path", "${workspaceFolder}/example.db"]
}
```

> [!NOTE]
> `.vscode/mcp.json` is tracked in this repo on purpose, since the
> example above has no machine-specific secrets. If you add a server
> that needs an API key or token, keep that one out of git instead.

### Building a Custom MCP Server: Weather

The first custom server in this repo, [weather_server.py](weather_server.py).
No official, no-key, zero-setup weather MCP server could be verified (see
below), so this became the first build-it-yourself example instead,
following the official MCP quickstart pattern.

1. Checked the community registry at `registry.modelcontextprotocol.io`
   for an existing weather server first. The results needed an API key,
   were an unverified third-party domain, or embedded a suspicious
   pay-per-query workflow with wallet/escrow instructions in the listing
   metadata, a prompt-injection-style red flag. None were suitable, so
   building one was the safer and more educational path.
2. Both dependencies, `mcp` and `httpx`, were already installed in
   `.venv` from the earlier `pip install mcp-server-fetch` step, so no
   extra install was needed.
3. Wrote `weather_server.py` with two tools, `get_alerts(state)` and
   `get_forecast(latitude, longitude)`, calling the free US National
   Weather Service API (`api.weather.gov`), which needs no API key.
4. Verified the tool functions directly, without going through the MCP
   protocol transport, by calling `get_forecast(38.9072, -77.0369)` in a
   plain Python snippet. It returned a real, live forecast for
   Washington, D.C., confirming the server logic works before wiring it
   into `.vscode/mcp.json`.
5. Registered it in `.vscode/mcp.json`:

   ```json
   "weather": {
     "command": "${workspaceFolder}/.venv/Scripts/python.exe",
     "args": ["${workspaceFolder}/weather_server.py"]
   }
   ```

6. Started it from the `Start` link above the `weather` entry, then
   asked the agent for a forecast and confirmed the `get_forecast` tool
   call was attributed to the `weather` server.

Unlike the four existing servers above, every line of `weather_server.py`
was written for this repo. That is the entire distinction between
existing and custom in practice.


