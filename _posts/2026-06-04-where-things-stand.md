---
layout: post
title: "Where Things Stand: Automated Pipeline and What's Next"
date: 2026-06-04
categories: groundhog
---

Six weeks in. This post is a checkpoint — what's been built, what's running automatically, and what's left before the project reaches its original learning goals.

---

## What's running

Groundhog is now a fully automated local data pipeline. Every day at 5pm MST a cron job fires:

```
ingestion/stocks.py → analytics/signals.py → analytics/alerts.py
```

- `stocks.py` fetches the latest OHLCV prices for every ticker in `watchlist.txt` via yfinance
- `signals.py` computes SMA(50), SMA(200), and Supertrend (daily + weekly) for each ticker
- `alerts.py` checks for direction flips and fires macOS notifications for buy/sell signals

BTC-USD runs 24/7 so the schedule runs every day including weekends. Traditional equity tickers just get no new data on weekends — `ON CONFLICT DO NOTHING` makes re-runs safe.

The cron entry:

```
TZ=America/Phoenix
0 17 * * * /Users/vvv/Projects/groundhog/scripts/daily_stocks.sh >> /Users/vvv/Projects/groundhog/data/logs/stocks.log 2>&1
```

Arizona (MST) doesn't observe daylight saving, so `America/Phoenix` is explicit and stable year-round.

---

## What's been built across the project

**Data ingestion**
- `health_metrics` — Garmin daily summary screenshots → steps, avg HR, active minutes
- `activities` — Garmin activity screenshots → workout type, distance, pace, HR
- `sleep_metrics` — Sleep8 screenshots → resting HR, HRV, breath rate, deep sleep, time to fall asleep
- `workouts` — SugarWOD daily WOD screenshots → name, category, structure type, full description
- `stock_watchlist` — yfinance OHLCV, daily, all tickers in watchlist

**Analytics**
- `stock_signals` — SMA cross and Supertrend (daily + weekly) per ticker per day
- `stock_alerts` — fired alert log with deduplication

**Agent**
- Text-to-SQL with tool calling (M2–M3)
- Persistent memory with DuckDB embeddings + vector similarity (M4)
- Multi-step reasoning: plan → execute → synthesize (M5)
- MCP server/client — tools exposed over stdio, any MCP client can connect (M5+)

---

## What's not done yet

**LangGraph**
The original learning plan listed LangGraph as the preferred framework for agentic workflows. The agent was built from scratch instead — which was the right call for learning fundamentals, but LangGraph is worth a dedicated spike. It handles state machines, conditional branching, and human-in-the-loop patterns that are hard to replicate cleanly with a hand-rolled loop.

**Advanced RAG**
M4 added naive RAG — embed a fact, store it, retrieve by cosine similarity. What's missing: entity-aware retrieval, proper chunking strategies, and evaluation of retrieval quality. The current memory is a flat list of facts with no structure. A more sophisticated RAG layer would understand that "INTC is bullish" and "Intel stock is trending up" are the same entity.

**Production hardening (M6)**
- Eval framework — measure vision extraction accuracy, agent answer quality
- Observability — log every tool call, latency, token count
- Prompt versioning — track which prompt produced which result
- Guardrails — catch bad SQL, hallucinated tickers, out-of-range values

---

## What's next

Two tracks in parallel:

**LangGraph spike** — rebuild the agent loop using LangGraph to understand where it adds value over a hand-rolled loop. The MCP server stays; only the client/reasoning layer changes.

**Advanced RAG** — add structure to the memory layer. Entity linking, better chunking, retrieval evaluation. The goal is making `recall()` actually useful for cross-source questions like "how does my sleep affect my workout performance?"

M6 production hardening comes after both — you need a stable agent before you instrument it.
