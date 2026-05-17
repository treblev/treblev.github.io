---
layout: post
title: "Day 8 — Grounding the LLM in DuckDB's SQL Dialect"
date: 2026-05-17
categories: groundhog
---

## The problem

First real query through the MCP client:

```
You: whats the average price of bitcoin over the last 7 days?

[tool] run_sql({'query': "SELECT AVG(closing_price) AS avg_price FROM stock_watchlist
       WHERE ticker = 'BTC-USD' AND date >= CURDATE() - INTERVAL 6 DAY;"})
       → SQL error: Catalog Error: Scalar Function with name curdate does not exist!
       Did you mean "current_date"?

[tool] run_sql({'query': "SELECT AVG(closing_price) AS avg_price FROM stock_watchlist
       WHERE ticker = 'BTC-USD' AND date >= CURRENT_DATE - INTERVAL '6 days';"})
       →  avg_price: 80352.6775

Agent: The average price of Bitcoin (BTC-USD) over the last 7 days is $80,352.68.
```

The model tried `CURDATE()` — MySQL syntax — got a DuckDB error, recovered with `CURRENT_DATE`, and eventually got the right answer. Two tool calls instead of one. Wasted round.

This is a predictable failure. LLMs are trained on large SQL corpora that skew toward MySQL and Postgres syntax. DuckDB uses ANSI SQL with its own conventions that differ in small but breaking ways. Without explicit guidance the model will guess, and the guess will often be wrong.

## Why a system prompt sentence isn't enough

The first instinct was to add a line to the system prompt:

```python
"The database is DuckDB. Use CURRENT_DATE (not CURDATE()), and INTERVAL '7 days' syntax (not INTERVAL 7 DAY).\n"
```

This works but it's the wrong place. The system prompt describes *behavior* — how the agent should act, when to call tools, how to handle memory. SQL dialect is *database documentation* — it belongs with the schema, not the instructions.

Mixing them means when you change the database engine, you have to edit behavioral instructions to find the dialect line. That's the wrong coupling.

## The fix: a dialect constant in the schema context

Both `agent/query.py` and `mcp_client/client.py` inject a schema string into the system prompt — table definitions, column types, actual ticker and activity type values. The dialect notes belong at the top of that string:

```python
_DUCKDB_DIALECT = """\
-- DuckDB dialect:
-- Dates:      CURRENT_DATE (not CURDATE()), CURRENT_TIMESTAMP (not NOW())
-- Intervals:  INTERVAL '7 days' (not INTERVAL 7 DAY)
-- Truncation: DATE_TRUNC('week', date), DATE_TRUNC('month', date)
-- Subqueries: no LIMIT inside subqueries — use a CTE instead
"""

def _get_schema(con):
    # ... build table definitions ...
    return _DUCKDB_DIALECT + "\n".join(parts)
```

The model sees it as part of the database documentation, right above the table definitions. The format — SQL comments — matches the context it appears in, so the model treats it as authoritative.

## What this looks like in context

The full schema string the model receives now starts with:

```
-- DuckDB dialect:
-- Dates:      CURRENT_DATE (not CURDATE()), CURRENT_TIMESTAMP (not NOW())
-- Intervals:  INTERVAL '7 days' (not INTERVAL 7 DAY)
-- Truncation: DATE_TRUNC('week', date), DATE_TRUNC('month', date)
-- Subqueries: no LIMIT inside subqueries — use a CTE instead

health_metrics(date DATE, steps INTEGER, avg_hr INTEGER, active_minutes INTEGER, ...)
stock_watchlist(date DATE, ticker VARCHAR, closing_price DECIMAL, ...)  -- tickers: INTC, BTC-USD
activities(...)  -- activity_types: running, walking, cardio, strength training
memory(id VARCHAR, fact TEXT, embedding FLOAT[], ...)
```

The dialect notes read naturally as comments on the schema. The model picks them up as constraints before generating any SQL.

## The broader principle

Where you put information shapes how the model weights it. Instructions in the system prompt get treated as behavioral rules. Information in a "database schema" block gets treated as technical documentation. Dialect notes are technical documentation — they belong with the schema, not the rules.

Same logic applies to other context you inject: ticker symbols and activity type values live in the schema as SQL comments (`-- tickers: INTC, BTC-USD`) because they're data constraints, not behavioral instructions.

If you ever swap DuckDB for Postgres, you change `_DUCKDB_DIALECT` in one place. The system prompt stays untouched.
