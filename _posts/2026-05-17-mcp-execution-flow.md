---
layout: post
title: "MCP Execution Flow"
date: 2026-05-17
categories: groundhog
---

## Converting Multi-step Reasoning to MCP

The multi-step reasoning agent built in M5 had all its tools as Python functions called in-process — `_run_sql()`, `_get_latest_price()`, etc. living alongside the agent loop in `agent/query.py`. MCP (Model Context Protocol) is Anthropic's open standard for separating tools from the reasoning layer. This post walks through converting that setup into an MCP server and a local MCP client.

**Why bother?** Two reasons. First, MCP is the industry standard — Claude Desktop, Cursor, and other AI hosts consume MCP servers natively. Second, the separation is architecturally cleaner: the server owns data access, the client owns reasoning. You can swap out either side independently.

---

## The architecture

**Before (M5):**

```
agent/query.py
├── tool definitions (TOOLS list)
├── tool implementations (_run_sql, _get_latest_price, ...)
└── agentic loop (_plan → execute → _synthesize)
```

Everything in one file. The loop calls Python functions directly.

**After (MCP):**

```
mcp_server/server.py   ← exposes tools over stdio (MCP protocol)
mcp_client/client.py   ← connects to server, drives Ollama for reasoning
```

The client and server communicate over **stdin/stdout** as a subprocess pair. The MCP SDK handles the JSON serialization; you write handlers.

---

## Building the MCP server

The server has three parts: initialization, tool registration, and tool execution.

**Initialization**

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server

server = Server("groundhog")
con = duckdb.connect(str(DB_PATH))
```

`Server("groundhog")` creates the server with a name. The DuckDB connection is opened once at startup and reused for all tool calls.

**Registering tools**

The `@server.list_tools()` decorator registers the handler that the client calls to discover what tools are available. Each tool is a `Tool` object with a name, description, and JSON schema for its arguments:

```python
from mcp.types import Tool

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="run_sql",
            description="Run a DuckDB SQL query and return the results.",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "A valid DuckDB SQL query."}
                },
                "required": ["query"],
            },
        ),
        Tool(
            name="get_latest_price",
            description="Get the latest closing price for a stock ticker.",
            inputSchema={
                "type": "object",
                "properties": {
                    "ticker": {"type": "string", "description": "Exact ticker symbol e.g. INTC, BTC-USD."}
                },
                "required": ["ticker"],
            },
        ),
        # ... get_recent_activities, get_health_summary, remember, recall
    ]
```

**Executing tools**

The `@server.call_tool()` decorator handles actual execution. It receives the tool name and arguments, routes to the right implementation, and returns the result as a `TextContent` object:

```python
from mcp.types import TextContent

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    result = _dispatch(name, arguments)
    return [TextContent(type="text", text=result)]
```

`_dispatch()` is just a routing function — same logic as `query.py` had before, now living in the server:

```python
def _dispatch(name: str, args: dict) -> str:
    if name == "run_sql":
        try:
            df = con.execute(args["query"]).fetchdf()
            return df.to_string(index=False) if not df.empty else "No results."
        except Exception as e:
            return f"SQL error: {e}"

    if name == "get_latest_price":
        row = con.execute(
            "SELECT date, closing_price FROM stock_watchlist WHERE ticker = ? ORDER BY date DESC LIMIT 1",
            [args["ticker"]],
        ).fetchone()
        return f"{args['ticker']} closing price on {row[0]}: ${row[1]:,.2f}" if row else "No data."

    # ... other tools
```

**Starting the server**

```python
async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, server.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

`stdio_server()` wires `stdin` and `stdout` as the communication channel. When you run this script, it blocks waiting for a client to connect.

---

## Building the MCP client

The client has three responsibilities: spawn the server, discover its tools, and drive the reasoning loop.

**Connecting to the server**

The client spawns the server as a subprocess and opens a `ClientSession` over its stdio:

```python
from mcp import ClientSession
from mcp.client.stdio import stdio_client, StdioServerParameters

params = StdioServerParameters(
    command=sys.executable,
    args=["mcp_server/server.py"],
)

async with stdio_client(params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
```

`session.initialize()` performs the MCP handshake — the client and server exchange capabilities.

**Discovering tools**

```python
mcp_tools = await session.list_tools()
```

This calls the server's `@server.list_tools()` handler over the wire. The client gets back `Tool` objects. But Ollama expects tools in its own format, so we convert:

```python
def _mcp_tool_to_ollama(tool) -> dict:
    return {
        "type": "function",
        "function": {
            "name": tool.name,
            "description": tool.description,
            "parameters": tool.inputSchema,
        },
    }

ollama_tools = [_mcp_tool_to_ollama(t) for t in mcp_tools.tools]
```

The schema is already JSON Schema — Ollama and MCP both use it, so the conversion is just a reshape.

**The agentic loop**

The reasoning loop is the same Plan → Execute → Synthesize structure from M5, but tool execution now goes over MCP instead of calling Python functions directly:

```python
# before (M5): direct Python call
result = _dispatch(con, name, args)

# after (MCP): call via protocol
mcp_result = await session.call_tool(name, args)
result = mcp_result.content[0].text
```

One line change. Everything else — planning, system prompt, fallback JSON parsing, synthesis — stays the same.

---

## Execution flow for a real question

Walking through "compare my workout frequency this week vs last week":

```
1. client spawns mcp_server/server.py as a subprocess
2. session.initialize() → MCP handshake over stdio
3. session.list_tools() → server returns 6 Tool objects
4. client converts to Ollama format

5. _plan() → Ollama produces a numbered plan (no tools called)
   [plan] 1. run_sql: count activities this week
          2. run_sql: count activities last week

6. Ollama decides to call run_sql
7. session.call_tool("run_sql", {"query": "SELECT COUNT(*) ..."})
   → JSON sent to server over stdin
   → server executes SQL against DuckDB
   → result sent back over stdout
   → client feeds result to Ollama

8. Ollama calls run_sql again for last week
   → same round-trip

9. Ollama has both results, stops calling tools
10. returns: "6 workouts this week, 0 last week"
```

The MCP protocol is just structured JSON messages over stdio. The SDK handles serialization on both ends — on the server side you write handlers, on the client side you `await` calls.

---

## Running it

```bash
python mcp_client/client.py
```

The client starts the server automatically as a subprocess. You don't run the server separately.

```
Groundhog MCP Client — 6 tools loaded via MCP. Type 'exit' to quit.

You: compare my workout frequency this week vs last week
  [plan] 1. run_sql: count this week  2. run_sql: count last week
  [tool] run_sql(...) → count_star(): 6
  [tool] run_sql(...) → count_star(): 0
Agent: You had 6 workouts this week compared to 0 last week.
```

---

## What changed vs M5

The reasoning logic is identical. The only structural change is where tool implementations live and how they're called:

| | M5 (`query.py`) | MCP |
|---|---|---|
| Tool implementations | Python functions in same file | `mcp_server/server.py` |
| Tool execution | Direct function call | `session.call_tool()` over stdio |
| Tool discovery | Hardcoded `TOOLS` list | `session.list_tools()` at runtime |
| Portability | CLI only | Any MCP-compatible client |

The separation means you can now point Claude Desktop (or any other MCP client) at the same `mcp_server/server.py` without changing a line of the server code.
