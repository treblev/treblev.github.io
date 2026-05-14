---
layout: post
title: "Day 4 — Stocks Ingestion + Agent Query Layer"
date: 2026-05-13
categories: groundhog
---

## What was built
- `ingestion/stocks.py` — yfinance-based stock price ingestion
- `config/watchlist.txt` — ticker + period config file (`INTC 7y`, `BTC-USD max`)
- `load_watchlist()` in `settings.py` — parses watchlist file into `(ticker, period)` tuples
- Historical backfill: 1,760 rows for INTC (2019–2026), 4,258 rows for BTC-USD (2014–2026)
- `agent/query.py` — text-to-SQL agent with interactive loop using `qwen2.5-coder:32b`

## Key decisions
- **yfinance** for stock data — free, no API key, good historical coverage
- **Per-ticker period in watchlist.txt** — avoids hardcoding periods in code, easy to extend
- **Two LLM calls in agent**: call 1 generates SQL, call 2 translates results to natural language answer
- **Schema injected into system prompt** — model knows table/column names at query time; enriched with actual ticker values and activity types so model doesn't hallucinate them
- **`ORDER BY date DESC LIMIT 1` rule** — told the model to use this for "latest price" queries instead of `CURRENT_DATE`, which fails when today's data hasn't been fetched yet

## Problems faced
- **Agent used wrong ticker symbol** — model generated `WHERE ticker = 'BTC'` instead of `'BTC-USD'`. Fixed by injecting actual ticker values from the DB into the schema context.
- **`CURRENT_DATE` returned no rows** — if today's price hasn't been fetched yet, the query returns empty. Added explicit instruction to use `ORDER BY date DESC LIMIT 1` for latest price queries.
- **DuckDB rowcount returns -1** — `result.rowcount` after a bulk INSERT returns -1 in DuckDB, not the actual inserted count. Cosmetic issue only; data was inserted correctly.

## Concepts touched
- Text-to-SQL agents
- Schema-as-context prompt engineering
- Prompt rules to constrain SQL generation
- Two-step LLM pipelines (SQL generation → natural language answer)
