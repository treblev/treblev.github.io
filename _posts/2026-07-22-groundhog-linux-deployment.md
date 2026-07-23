---
layout: post
title: "Moving Groundhog to a Linux Box"
date: 2026-07-22
categories: groundhog
---

Groundhog started as a Mac-local personal data pipeline: DuckDB on disk,
Python scripts for ingestion, Ollama for local AI, and a daily stock job kicked
off by the desktop environment. The next step was moving the runtime to a small
Linux box while keeping the privacy model intact.

The goal was not to turn Groundhog into a cloud app. The goal was simpler:
make it run unattended on a local machine, keep personal data local, and let
OpenClaw handle chat and delivery.

---

## The target shape

The boundary after the migration is:

```
OpenClaw
  chat, scheduling, delivery, model/tool orchestration
        |
        v
Groundhog MCP server
  local tools over personal data
        |
        v
Groundhog core
  DuckDB, ingestion, analytics, alerts, memory
```

Groundhog decides what happened. OpenClaw decides how to talk about it.

That boundary matters. If the data pipeline and the chat harness both try to
own scheduling, notifications, memory, and state, the system becomes hard to
debug. Groundhog should stay deterministic and auditable. OpenClaw should be
the conversational layer.

---

## Linux layout

The Linux deployment uses a dedicated service user. The repo, virtual
environment, systemd user services, and DuckDB file all live under that user's
ownership.

The important choices:

- run the stock pipeline as an unprivileged service user
- keep the DuckDB file outside the git repo
- keep systemd services as user services, not root services
- enable linger so user timers continue after logout
- keep Ollama local on the LAN
- keep OpenAI/Anthropic fallback disabled for personal data

The stock pipeline itself did not change conceptually:

```
ingestion/stocks.py -> analytics/signals.py -> analytics/alerts.py
```

What changed was how it runs and how it reports failure.

---

## systemd user timer

On macOS this had been a desktop/local script workflow. On Linux it became a
systemd user timer.

The install flow is:

```bash
mkdir -p ~/.config/systemd/user
cp deploy/systemd/user/groundhog-stocks.service ~/.config/systemd/user/
cp deploy/systemd/user/groundhog-stocks.timer ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now groundhog-stocks.timer
```

The timer is pinned to `America/Phoenix` and runs after market close. The
service calls `scripts/daily_stocks.sh`, which prints clear section markers:

```bash
--- Fetching prices ---
--- Computing signals ---
--- Checking alerts ---
```

The useful operational commands are:

```bash
systemctl --user status groundhog-stocks.timer
systemctl --user list-timers groundhog-stocks.timer
systemctl --user start groundhog-stocks.service
journalctl --user -u groundhog-stocks.service -n 100 --no-pager
```

That last command became the main feedback loop. If the job fails, the journal
should say where.

---

## Linux-specific alert behavior

The old alert path assumed a Mac desktop notification. That does not make sense
on a headless Linux service user.

The fix was to make alert delivery configurable:

```
GROUNDHOG_ALERT_BACKEND=none
```

With that setting, `analytics/alerts.py` still records deduped rows in
`stock_alerts`, but it does not try to fire a desktop notification. That is the
right split for the new architecture: Groundhog records the fact; OpenClaw can
decide later whether and how to deliver it.

---

## Dependency issue

The Linux venv exposed a dependency conflict:

```
yfinance -> websockets >= 13.0
langgraph-sdk -> websockets < 16
requirements.txt -> websockets == 16.0
```

The fix was to pin `websockets` to a compatible version instead of fighting the
resolver. After that, the Linux venv installed cleanly.

This is the boring kind of migration bug, but it is exactly why the migration
needed to happen on the target machine instead of being assumed from the Mac.

---

## yfinance fetching too much history

The first stock job worked, but the logs showed an obvious waste:

```
500 rows fetched, 1 inserted.
```

The script was idempotent because DuckDB ignored duplicate `(date, ticker)`
rows, but idempotent does not mean efficient. The fix was to check the latest
stored date per ticker and fetch only from the next missing date.

There was one more catch: yfinance can still return a larger window than
expected. So the ingestion code now also filters the returned dataframe to the
requested date range before insert.

The regression test deliberately mocks the annoying case: Yahoo returns 500
rows, but only one row belongs in the requested window. The test failed before
the fix and passes now.

---

## Smoke tests

The migration produced two kinds of tests.

Offline regression tests:

```bash
venv/bin/python -m unittest discover -s tests -p 'test_*.py' -v
```

Live smoke tests:

```bash
venv/bin/python tests/smoke_test.py
```

The offline suite checks deterministic behavior without touching the real DB or
network. The smoke test imports the real modules, calls the live DB tool layer,
and verifies memory write/read behavior through local Ollama embeddings.

That split is useful. Regression tests prove specific bugs do not come back.
Smoke tests prove the deployed box is wired correctly.

---

## Ollama URL cleanup

The most confusing failure was memory smoke tests failing with:

```
[Errno 111] Connection refused
```

Ollama was reachable from the Linux box, but the memory code still had a
hardcoded localhost embeddings URL. The quick workaround was `OLLAMA_HOST`, but
that made the smoke test too easy to run incorrectly.

The final fix was to put the Ollama base URL in `config/settings.py`:

```python
OLLAMA_BASE_URL = "http://<ollama-host>:11434"
OLLAMA_CHAT_URL = f"{OLLAMA_BASE_URL}/api/chat"
OLLAMA_EMBEDDINGS_URL = f"{OLLAMA_BASE_URL}/api/embeddings"
```

Then the direct HTTP clients, memory embeddings, legacy MCP client, and
LangGraph client all read from the same source of truth.

No more per-command `OLLAMA_HOST`.

---

## What Phase 0 means

At the end of this migration phase:

- the Linux repo pulls cleanly
- the venv installs
- stock ingestion is incremental
- alerts are database-first on Linux
- systemd owns the stock schedule
- Ollama access is explicit in config
- offline regression tests pass
- live smoke tests pass

That is enough to call the Linux migration stable.

The next phase is not "add more AI." The next phase is service state:

```
agent_runs
events
outbox
```

Those tables will make Groundhog observable as a long-running personal data
service before it becomes more agentic. The lesson from this migration is the
same as most production work: boring state and clear boundaries beat clever
automation.
