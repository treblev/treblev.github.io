---
layout: post
title: "Stock Technical Analysis Signals and macOS Alerts"
date: 2026-06-03
categories: groundhog
---

Groundhog already stores daily stock prices via yfinance. This post adds the next layer: computing SMA and Supertrend signals from that price data, detecting when they flip direction, and firing macOS notifications for buy/sell alerts — all locally, no external services.

---

## The pipeline

Three scripts, chained in order by a cron job at 5pm Monday–Friday:

```
ingestion/stocks.py → analytics/signals.py → analytics/alerts.py
```

Each script is independent and can be run manually. The cron job just automates the chain after market close.

---

## Schema changes

`stock_watchlist` previously only stored closing price. Supertrend requires High and Low to compute ATR, so the table was expanded to full OHLCV:

```sql
ALTER TABLE stock_watchlist ADD COLUMN IF NOT EXISTS open DECIMAL(10, 2);
ALTER TABLE stock_watchlist ADD COLUMN IF NOT EXISTS high DECIMAL(10, 2);
ALTER TABLE stock_watchlist ADD COLUMN IF NOT EXISTS low DECIMAL(10, 2);
ALTER TABLE stock_watchlist ADD COLUMN IF NOT EXISTS volume BIGINT;
```

Two new tables:

```sql
CREATE TABLE stock_signals (
    id VARCHAR PRIMARY KEY,   -- hash of date|ticker|signal_type|timeframe
    date DATE,
    ticker VARCHAR,
    signal_type VARCHAR,      -- "sma_cross" or "supertrend"
    timeframe VARCHAR,        -- "daily" or "weekly"
    value DECIMAL(10, 4),     -- SMA gap or supertrend line price
    direction VARCHAR         -- "bullish" or "bearish"
);

CREATE TABLE stock_alerts (
    id VARCHAR PRIMARY KEY,   -- hash of date|ticker|alert_type
    date DATE,
    ticker VARCHAR,
    alert_type VARCHAR,
    message VARCHAR,
    notified_at TIMESTAMP
);
```

`stock_alerts` is the deduplication table — once an alert fires for a given date/ticker/type, it won't fire again even if the script re-runs.

---

## Signals

`analytics/signals.py` computes three signals per ticker per run and writes them to `stock_signals`:

**SMA cross (daily)**
- SMA(50) and SMA(200) computed on closing price
- `direction = "bullish"` when SMA50 > SMA200 (golden cross territory), `"bearish"` when below
- `value` = SMA50 − SMA200 (the gap; zero crossing is the event)

**Supertrend (daily and weekly)**
- Standard params: period=10, multiplier=3.0 — same defaults as the popular Pine Script indicator
- For weekly: daily OHLCV is resampled to weekly (Friday close, weekly high/low) before computing
- `direction = "bullish"` when price is above the supertrend line, `"bearish"` when below
- `value` = the supertrend line price level

---

## Getting Supertrend right

The first implementation used pandas EWM with `span=period`, which gives `alpha = 2/(period+1)`. Pine Script's `atr()` uses Wilder's RMA with `alpha = 1/period` — a slower-decaying smoothing that produces meaningfully different ATR values.

The band clamping logic was also wrong. Supertrend tracks two bands independently:
- **Lower band** (bullish line) only ratchets **up** — it never drops while price stays above it
- **Upper band** (bearish line) only ratchets **down** — it never rises while price stays below it

The corrected implementation:

```python
# Wilder's RMA — matches Pine Script atr() default
atr = tr.ewm(alpha=1 / period, adjust=False).mean()

for i in range(1, len(df)):
    prev_close = close.iloc[i - 1]

    # Ratchet lower band up, upper band down
    lower.iloc[i] = max(raw_lower.iloc[i], lower.iloc[i-1]) if prev_close > lower.iloc[i-1] else raw_lower.iloc[i]
    upper.iloc[i] = min(raw_upper.iloc[i], upper.iloc[i-1]) if prev_close < upper.iloc[i-1] else raw_upper.iloc[i]

    # Flip on previous bar's band value (matches Pine's up1/dn1)
    if prev_dir == -1 and close.iloc[i] > upper.iloc[i-1]:
        direction.iloc[i] = 1
    elif prev_dir == 1 and close.iloc[i] < lower.iloc[i-1]:
        direction.iloc[i] = -1
    else:
        direction.iloc[i] = prev_dir
```

---

## Alerts

`analytics/alerts.py` reads the last two rows per ticker per signal type and checks if direction changed:

```python
rows = con.execute("""
    SELECT date, direction FROM stock_signals
    WHERE ticker = ? AND signal_type = 'supertrend' AND timeframe = 'daily'
    ORDER BY date DESC LIMIT 2
""", [ticker]).fetchall()

if rows[0][1] != rows[1][1]:  # direction flipped
    _notify("Groundhog Stock Alert", f"{ticker}: Supertrend flipped BULLISH")
```

Four alert types:
- `golden_cross` — SMA50 crosses above SMA200
- `death_cross` — SMA50 crosses below SMA200
- `supertrend_daily_bullish` / `supertrend_daily_bearish` — daily Supertrend flip
- `supertrend_weekly_bullish` / `supertrend_weekly_bearish` — weekly Supertrend flip

macOS notification via `osascript`:

```python
subprocess.run(["osascript", "-e", f'display notification "{message}" with title "Groundhog Stock Alert"'])
```

---

## Cron setup

`scripts/daily_stocks.sh` chains the three scripts and logs output:

```bash
#!/bin/bash
set -e
cd "$(dirname "$0")/.."
PYTHON=venv/bin/python

$PYTHON ingestion/stocks.py
$PYTHON analytics/signals.py
$PYTHON analytics/alerts.py
```

Crontab entry (5pm Mon–Fri):

```
0 17 * * 1-5 /Users/vvv/Projects/groundhog/scripts/daily_stocks.sh >> /Users/vvv/Projects/groundhog/data/logs/stocks.log 2>&1
```

---

## Interpreting the signals table

| signal_type | timeframe | value | direction |
|---|---|---|---|
| `sma_cross` | `daily` | SMA50 − SMA200 gap | `bullish` / `bearish` |
| `supertrend` | `daily` | supertrend line price | `bullish` / `bearish` |
| `supertrend` | `weekly` | supertrend line price | `bullish` / `bearish` |

Alerts only fire on **direction flips** — a signal that stays bullish for weeks generates no noise. The `stock_alerts` table records every fired alert so re-runs are idempotent.
