---
layout: post
title: "LangGraph Agent: From Manual Graph to create_agent()"
date: 2026-06-27
categories: groundhog
---

The last post left off with `plan_node` wired into a LangGraph `StateGraph`. This post covers how that approach got scrapped, what replaced it, and the most useful question that came up along the way.

---

## The manual graph felt like vibe coding

After building `AgentState`, `build_graph()`, `plan_node`, and `llm_node` step by step, something felt off. The code was growing — conditional edges, tool routing logic, MCP session passed as a closure — and it was starting to feel like copying patterns without really understanding them.

The right call was to stop and actually learn LangGraph properly. I built a toy agent from scratch — no DuckDB, no MCP, just a fake `get_weather()` tool and a simple graph. Once the mechanics clicked (state flows between nodes as dicts, `add_messages` appends rather than overwrites, the conditional edge is just a function that returns a node name), coming back to Groundhog felt like recognition rather than mystery.

---

## Does LangGraph replace MCP?

This was the most useful question from the session, so it's worth writing down clearly.

**No. They do completely different things.**

**MCP owns the tools.** `run_sql`, `get_latest_price`, `remember`, `recall` — these live in `mcp_server/server.py` and are the interface between the agent and the database. Nothing about that changes.

**LangGraph owns the control flow.** It's the loop that decides when to call a tool, passes results back to the LLM, and knows when to stop. In the hand-rolled `mcp_client/client.py`, that logic was 120 lines of imperative Python. LangGraph (or in this case `create_agent()`) replaces that with a managed framework.

The stack looks like this:

```
create_agent()
  └── tool wrappers (plain Python async functions)
        └── session.call_tool()
              └── mcp_server/server.py
                    └── DuckDB
```

MCP is still the stable tool layer. `create_agent()` just replaced the messy loop wrapped around it.

---

## Switching to create_agent()

LangChain 1.3 ships `create_agent()` which handles the LLM + tool routing loop in one call:

```python
from langchain.agents import create_agent

agent = create_agent(
    model="ollama:qwen3:32b",
    tools=tools,
    system_prompt="...",
)
result = await agent.ainvoke({"messages": [{"role": "user", "content": question}]})
```

That's it. No `StateGraph`, no nodes, no edges, no conditional routing. Everything we built manually in the spike — `AgentState`, `build_graph()`, `plan_node`, `llm_node`, `tool_node` — gone.

The model string format `"ollama:<model>"` is how you point it at a local Ollama instance instead of a cloud API. Personal data stays local.

---

## Wrapping MCP tools as Python functions

`create_agent()` expects plain Python functions with docstrings. MCP tools live behind a stdio process. The bridge is a set of thin async wrapper functions that close over the MCP session:

```python
def _make_tools(session: ClientSession) -> list:
    async def run_sql(query: str) -> str:
        """Run a SQL query against the local DuckDB database."""
        result = await session.call_tool("run_sql", {"query": query})
        return result.content[0].text if result.content else "No result."

    async def get_latest_price(ticker: str) -> str:
        """Get the latest closing price for a stock ticker."""
        result = await session.call_tool("get_latest_price", {"ticker": ticker})
        return result.content[0].text if result.content else "No result."

    # ... recall, remember, get_health_summary, get_recent_activities

    return [run_sql, get_latest_price, ...]
```

The docstring on each function is what `create_agent()` passes to the LLM as the tool description — same role as the MCP tool's `description` field in the old client. The async is needed because `session.call_tool()` is a coroutine and the whole `run()` function is already async.

`_make_tools()` is called once after the MCP session starts, so the session reference is baked into the closures for the lifetime of the process.

---

## Schema context

The schema builder from `mcp_client/client.py` was ported unchanged. It runs at startup, queries `SHOW TABLES` and `DESCRIBE <table>` for each table, and adds domain hints for tables where column names alone aren't enough:

```python
elif table == "stock_signals":
    note = "  -- signal_type: 'sma_cross' or 'supertrend'; timeframe: 'daily' or 'weekly'; direction: 'bullish' or 'bearish'"
```

The full schema string gets injected into the system prompt so the LLM always knows what it's working with.

---

## What actually changed

The chat REPL was already in `mcp_client/client.py` — that's not new. What changed is what runs underneath:

| | `mcp_client/client.py` | `langgraph_client/client.py` |
|---|---|---|
| Tool routing | Hand-rolled for-loop | `create_agent()` |
| Fallback JSON parser | Yes (lines 91–101) | Not needed |
| Planning step | Explicit separate LLM call | Dropped for now |
| Lines of code | ~190 | ~110 |

The fallback JSON parser in the old client (the block that used regex to extract tool calls from raw content when Ollama didn't format them correctly) is gone. `create_agent()` handles that internally.

The planning step was dropped. It can come back later as middleware — a pre-processing step before `agent.ainvoke()` — which is cleaner than baking it into the graph anyway.

---

## Running it

```bash
venv/bin/python langgraph_client/client.py
```

```
Groundhog Agent — 6 tools loaded. Type 'exit' to quit.

You: What is the latest closing price for AAPL?
Agent: The latest closing price for AAPL is $211.17 (as of 2026-06-26).
```
