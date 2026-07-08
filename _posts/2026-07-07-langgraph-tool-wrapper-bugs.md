---
layout: post
title: "LangGraph Client: The Tool Wrappers Were Lying"
date: 2026-07-07
categories: groundhog
---

The [last post]({% post_url 2026-06-27-langgraph-create-agent %}) ended on a confident note: `create_agent()` replaced ~80 lines of hand-rolled tool-routing with one function call, and even claimed the old client's fallback JSON parser was "not needed" anymore. Running a real side-by-side test against the old client found out that claim was only half true — and surfaced three tool wrappers that were quietly broken since the rewrite.

---

## The test

Four questions, run against both `mcp_client/client.py` (old, hand-rolled loop) and `langgraph_client/client.py` (new, `create_agent()`), against the live DuckDB:

1. AAPL's latest closing price
2. Whether AAPL had a recent golden/death cross
3. Most recent workout + category
4. Latest health summary

Questions 1–3 were a wash or a win for the new client — it was 2–3x faster on every multi-tool question, and it actually fixed a bug in the old client, which had confused `day_of_week` with `category` in its own SQL. Question 4 broke it.

## `get_health_summary()` was calling a tool that doesn't exist

The wrapper in `langgraph_client/client.py` looked like this:

```python
async def get_health_summary() -> str:
    """Get a summary of recent health metrics."""
    result = await session.call_tool("get_health_summary", {})
    return result.content[0].text if result.content else "No result."
```

Zero arguments. But the actual MCP tool, defined in `mcp_server/server.py`, is a single-date lookup:

```python
Tool(
    name="get_health_summary",
    description="Get the health metrics (steps, avg HR, active minutes) for a specific date.",
    inputSchema={
        "type": "object",
        "properties": {"date": {"type": "string", "description": "Date in YYYY-MM-DD format."}},
        "required": ["date"],
    },
),
```

Every call failed server-side with `Input validation error: 'date' is a required property`. The model would sometimes recover by falling back to `run_sql` on its own, and sometimes just gave up and printed the raw failed tool-call as its final answer to the user — which is the "not needed anymore" fallback parser biting back. `create_agent()` doesn't have an equivalent to the old client's regex-based JSON-extraction shim, so when the local model (`qwen3:32b`) occasionally emits a tool call as plain text instead of a structured `tool_calls` entry, nothing catches it. Last post's claim was wrong on that point — noting it here so it doesn't get repeated.

Two more wrappers had the same class of bug, just less visible because nothing in the test set happened to exercise them:

```python
# before — wrong keys entirely
async def remember(key: str, value: str) -> str:
    result = await session.call_tool("remember", {"key": key, "value": value})

async def recall(key: str) -> str:
    result = await session.call_tool("recall", {"key": key})
```

The server expects `fact` for `remember`, and `query`/`top_k` for `recall`. Both would have thrown a `KeyError` server-side the moment either tool actually got called.

## Why this didn't happen to the old client

`mcp_client/client.py` never hand-writes tool contracts. It converts the MCP server's own `Tool` objects straight into Ollama's function-calling format:

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
```

There's no second copy of the contract to drift out of sync. `create_agent()` wants plain Python functions with docstrings instead of raw JSON schemas, so the LangGraph rewrite introduced a second, hand-maintained copy of every tool's signature — and three of the six drifted immediately.

## The fix

Match the wrapper signatures to what the server actually requires:

```python
async def get_health_summary(date: str) -> str:
    """Get health metrics (steps, avg HR, active minutes) for a specific date (YYYY-MM-DD).
    Use run_sql first to find the latest available date if needed."""
    result = await session.call_tool("get_health_summary", {"date": date})
    return result.content[0].text if result.content else "No result."

async def remember(fact: str) -> str:
    """Save a fact or preference to persistent memory for future recall."""
    result = await session.call_tool("remember", {"fact": fact})
    return result.content[0].text if result.content else "Saved."

async def recall(query: str, top_k: int = 3) -> str:
    """Search persistent memory for the user's personal opinions, preferences, and stated beliefs."""
    result = await session.call_tool("recall", {"query": query, "top_k": top_k})
    return result.content[0].text if result.content else "Nothing found."
```

Verified each one directly against the live MCP session afterward — `get_health_summary` now correctly forces the model through a two-step "find the date, then look it up" pattern instead of erroring, and `remember`/`recall` round-trip cleanly.

Also added a line to the system prompt nudging the agent to check related tables (`sleep_metrics`, `workouts`) when the obvious one comes back empty, since the old client happened to get that behavior for free from its separate planning step, which the new client dropped. It only half-sticks on a local 32B model — worth revisiting once there's a system prompt logging/eval setup, part of the M6 hardening work still on the roadmap.

## Takeaway

Hand-written tool wrappers are a second source of truth. Any time the MCP server's tool contracts change, both `mcp_server/server.py` and the wrapper docstrings in `langgraph_client/client.py` need to move together — nothing enforces that today. A small script that diffs the MCP `Tool.inputSchema` against the wrapper's Python signature at startup would catch this class of bug before it reaches a user question instead of after.
