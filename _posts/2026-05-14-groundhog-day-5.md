---
layout: post
title: "Day 5 — Agentic Tool Calling"
date: 2026-05-14
categories: groundhog
---

## Agentic Tool Calling

The agent was rewritten from a two-call SQL pipeline into a proper agentic loop with tool calling.

**How it works:**

Instead of always generating SQL, the model is given a set of tools and decides which one to call:

- `run_sql(query)` — execute arbitrary SQL against DuckDB
- `get_latest_price(ticker)` — latest closing price for a ticker
- `get_recent_activities(limit)` — last N workout activities
- `get_health_summary(date)` — steps, HR, active minutes for a date

The loop runs up to 5 rounds — model calls a tool, we execute it, feed the result back, model decides to call another tool or give a final answer.

With tool calling, the model chooses the right tool for the question. `get_latest_price("BTC-USD")` always works correctly regardless of how you phrase the question — the implementation handles the SQL. It also opens the door to tools that aren't SQL at all (fetch a live price, call an API, read a file).

**Problem: Qwen's special tokens leaking into output**

`qwen2.5-coder:32b` doesn't use the structured `tool_calls` field in the response — instead it outputs the tool call as JSON text in `content`. On top of that, it leaks its internal template tokens into the output:

```
{"name": "get_recent_activities", "arguments": {}}<|im_start|><|im_start|>
```

The `<|im_start|>` tokens are Qwen's chat template markers that should never appear in the final output — but they bleed through via Ollama. This caused `json.loads()` to fail, so the fallback parser didn't trigger and the raw JSON was returned as the answer.

Fix: use greedy regex `\{.*\}` to extract just the JSON object, ignoring anything after the last `}`:

```python
match = re.search(r'\{.*\}', content, re.DOTALL)
```

Greedy matching ensures the full nested JSON (including `"arguments": {"query": "..."}`) is captured, not just the innermost `{}`.
