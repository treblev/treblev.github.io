---
layout: post
title: "Replacing the Hand-Rolled Agent Loop with LangGraph"
date: 2026-06-15
categories: groundhog
---

The agent in Groundhog started life as a hand-rolled loop in `mcp_client/client.py`. It worked, but it was a tangle of manual state management, explicit tool routing, and fallback JSON parsing. This post covers why I'm replacing it with LangGraph, what the old loop actually does, and where the spike is at.

---

## Why LangGraph?

The first question was obvious: doesn't the LLM do the reasoning? Why do I need a graph framework on top?

The short answer: the LLM decides *what to do*, but something else has to *run the loop* — dispatch tool calls, pass results back, decide when to stop. In the hand-rolled version, that something else is 120 lines of imperative Python. LangGraph replaces that imperative logic with a declared graph of nodes and edges, and gives you state management, conditional routing, and the ability to pause and inspect mid-run for free.

The segregation between LangGraph and MCP is clean: MCP owns the *tools* (the things the agent can call), LangGraph owns the *control flow* (the order it calls them and what happens between calls).

---

## What the old loop actually does

The hand-rolled agent lives in `mcp_client/client.py`. There are two core functions.

**`_chat(messages, tools)`** is a thin wrapper around the Ollama HTTP API. It takes a message list and optional tool definitions, posts to `localhost:11434/api/chat`, and returns the raw message dict. No state, no logic — just one HTTP call.

**`_ask(question, session, ollama_tools, schema)`** is where the agent actually lives:

1. **Planning step**: calls `_chat` with a system prompt that says "write a plan, do not call tools." Stores the plan text.
2. **Tool loop**: calls `_chat` again with tools bound and the plan injected into the system prompt. Checks the response for `tool_calls`. If present, dispatches each call via the MCP session, appends the tool result as a `role: tool` message, and loops. Runs up to `MAX_TOOL_ROUNDS = 8` times.
3. **Synthesis fallback**: if the round limit hits, sends one final no-tools call with "give a direct answer based on what you have."

The most interesting part is lines 91–101: a fallback JSON parser for when Ollama doesn't format tool calls correctly. Instead of returning a structured `tool_calls` array, the model sometimes buries a raw JSON blob inside the `content` field. The fallback uses a regex to extract it, checks if the `name` field matches a known tool, and reconstructs the call manually. It's a workaround for model inconsistency, and it's the kind of thing you don't want to be maintaining by hand.

---

## What LangGraph replaces

The tool routing logic — "does this response have tool calls? dispatch them, loop back, or stop?" — becomes a conditional edge in the graph. Instead of a for-loop with break conditions, you get:

```
START → plan → llm → [tool calls?] → tool → llm → ... → END
                           ↓ no
                          END
```

The fallback JSON parser still needs to exist somewhere, but it becomes a node responsibility rather than something woven through the main loop.

Pausing and inspecting is a first-class LangGraph feature: you can add a `interrupt_before` on any node and the graph suspends, lets you inspect `state`, and resumes. The hand-rolled loop has no equivalent — you'd have to add print statements and re-run.

---

## Current state of the spike

`langgraph_client/client.py` has three things so far:

**`AgentState`** — a `TypedDict` that is the graph's shared memory: `question`, `messages` (append-only via `add_messages`), `plan`, `rounds`, `answer`.

**`ChatOllama`** — LangChain's wrapper around the Ollama API, initialized at module level with the model from `config/settings.py`.

**`build_graph(schema, tool_names)`** — a factory function that closes over the schema and tool list (known only after MCP starts), defines nodes as closures, and returns a compiled graph. Currently only `plan_node` exists inside it: calls the LLM with the planning prompt, writes the result to `state.plan`.

```python
def plan_node(state: AgentState) -> dict:
    response = llm.invoke([
        SystemMessage(content=(
            "You are a planning assistant. Write a concise numbered plan..."
            f"\n\nAvailable tools: {', '.join(tool_names)}"
            f"\n\nDatabase schema:\n{schema}"
        )),
        HumanMessage(content=f"Plan how to answer: {state['question']}"),
    ])
    return {"plan": response.content.strip()}
```

The graph is wired `START → plan → END` for now — a temporary edge so the graph compiles and is testable at each step.

Steps 4–7 still to build: `llm_node` (LLM with tools bound), `tool_node` (MCP dispatch), conditional edges, schema builder ported from the old client, and a side-by-side test against the hand-rolled loop.
