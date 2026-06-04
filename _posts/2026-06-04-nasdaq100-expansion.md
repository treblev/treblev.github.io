---
layout: post
title: "Expanding to Nasdaq-100: Watchlist Automation and Agent Context"
date: 2026-06-04
categories: groundhog
---

Three things shipped today: the watchlist expanded from 6 handpicked tickers to the full Nasdaq-100, the agent got richer schema context so it can query signals and workouts correctly, and a DuckDB rowcount bug got fixed along the way.

---

## Nasdaq-100 watchlist automation

`scripts/update_watchlist.py` scrapes the current Nasdaq-100 components from Wikipedia and merges them into `config/watchlist.txt`, preserving any custom periods already set.

```python
def fetch_nasdaq100() -> set[str]:
    headers = {"User-Agent": "Mozilla/5.0 (compatible; groundhog-watchlist/1.0)"}
    response = httpx.get("https://en.wikipedia.org/wiki/Nasdaq-100", headers=headers)
    soup = BeautifulSoup(response.text, "lxml")
    table = soup.find("table", {"id": "constituents"})
    tickers = set()
    for row in table.find_all("tr")[1:]:
        cells = row.find_all("td")
        if cells:
            tickers.add(cells[0].get_text(strip=True))
    return tickers
```

Two things worth noting:

**Wikipedia blocks urllib.** `pd.read_html()` uses urllib under the hood and gets a 403. `httpx` with a browser-style User-Agent header gets through fine.

**Column order matters.** The Wikipedia Nasdaq-100 table has `Ticker` as the first column (`cells[0]`), not the second. Getting this wrong silently writes company names as tickers — which look valid until you try to fetch prices and yfinance returns nothing.

The merge logic keeps existing entries untouched:

```python
for ticker in nasdaq100:
    if ticker not in merged:
        merged[ticker] = DEFAULT_PERIOD  # "2y"
```

So INTC stays `7y`, BTC-USD stays `max`, MSFT stays `10y`. Only genuinely new tickers get the `2y` default. Tickers that rotate out of the Nasdaq-100 stay in the watchlist — you might still want to track them.

Run it whenever you want to pick up new components:

```bash
venv/bin/python scripts/update_watchlist.py
```

Output:
```
Fetching Nasdaq-100 components from Wikipedia...
  Found 101 tickers
  Existing watchlist: 6 tickers
  Added 99 new tickers: AAPL, ABNB, ADBE, ...
  Total: 105 tickers
Done. Run ingestion/stocks.py to backfill new tickers.
```

---

## Backfilling 105 tickers

After updating the watchlist, running `ingestion/stocks.py` fetches 2 years of OHLCV for all new tickers. 105 tickers × ~502 trading days each = ~52,000 new rows in `stock_watchlist`. Takes a couple of minutes, runs silently, and is safe to re-run:

```
Fetching AAPL (2y)...
  502 rows fetched, 502 inserted.
Fetching BTC-USD (max)...
  4279 rows fetched, 0 inserted.   ← already up to date
Fetching INTC (7y)...
  1761 rows fetched, 1 inserted.   ← one new day since last run
```

---

## DuckDB rowcount bug

`ON CONFLICT DO NOTHING` in DuckDB returns `rowcount = -1` for skipped rows instead of `0`. The original code was summing `result.rowcount` directly, producing outputs like `-1760 inserted` for INTC — wrong for existing tickers, and also wrong for new tickers where every row should have counted.

The fix: compare row counts before and after the bulk insert instead of trusting `rowcount`:

```python
before = con.execute("SELECT COUNT(*) FROM stock_watchlist WHERE ticker = ?", [ticker]).fetchone()[0]
# ... insert all rows ...
after = con.execute("SELECT COUNT(*) FROM stock_watchlist WHERE ticker = ?", [ticker]).fetchone()[0]
return after - before
```

Now `0 inserted` means all rows already existed, and a positive number means new rows actually landed.

---

## Default period changed to 2y

Previously, a ticker in `watchlist.txt` without an explicit period defaulted to `"1d"` — just yesterday's price. That's not enough data for SMA200 (which needs 200+ rows), so signals would silently skip the ticker.

Changed to `"2y"`, which gives ~500 trading days — enough headroom for all three signals.

---

## Agent schema context enrichment

The MCP client builds a schema string at startup that gets injected into every prompt. Previously it looked like this for signals:

```
stock_signals(id VARCHAR, date DATE, ticker VARCHAR, signal_type VARCHAR, timeframe VARCHAR, value DECIMAL, direction VARCHAR)
```

The agent could see the column names but had no idea what values were valid. It might write `WHERE signal_type = 'bullish'` when it should write `WHERE direction = 'bullish' AND signal_type = 'supertrend'`.

Now each table gets a comment with its domain knowledge:

```python
elif table == "stock_signals":
    note = "  -- signal_type: 'sma_cross' or 'supertrend'; timeframe: 'daily' or 'weekly'; direction: 'bullish' or 'bearish'; value: SMA gap (sma_cross) or supertrend line price (supertrend)"
elif table == "stock_alerts":
    note = "  -- alert_type: 'golden_cross', 'death_cross', 'supertrend_daily_bullish', 'supertrend_daily_bearish', 'supertrend_weekly_bullish', 'supertrend_weekly_bearish'"
elif table == "workouts":
    # query actual values from DB dynamically
    types = ...  # SELECT DISTINCT structure_type FROM workouts
    cats = ...   # SELECT DISTINCT category FROM workouts
    note = f"  -- structure_types: {', '.join(types)}; categories: {', '.join(cats)}"
```

`workouts` hints are queried dynamically at startup — so as new structure types or categories land in the DB, the agent picks them up automatically without any code changes.

---

## What happens when you run alerts

`analytics/alerts.py` reads the last two signal rows per ticker per signal type and checks if direction changed. Three possible outcomes:

**No alerts fire** — all tickers have the same direction as yesterday. The script prints a line per ticker and exits. This is the most common case.

**Alerts fire** — one or more tickers flipped. For each flip:
- A macOS notification appears
- A row is written to `stock_alerts` (prevents re-firing on the next run)
- Console output: `Alert fired: NVDA: Supertrend (daily) flipped BULLISH (BUY signal)`

Six alert types can fire: `golden_cross`, `death_cross`, `supertrend_daily_bullish`, `supertrend_daily_bearish`, `supertrend_weekly_bullish`, `supertrend_weekly_bearish`.

**No signals found** — `stock_signals` is empty, meaning `signals.py` hasn't been run yet:
```
No signals found. Run analytics/signals.py first.
```

The `stock_alerts` table is the deduplication layer — once an alert fires for a given date/ticker/type, it won't fire again even if the pipeline re-runs multiple times that day.
