---
layout: post
title: "Adding Sleep Data with Sleep8 Screenshots"
date: 2026-05-22
categories: groundhog
---

Groundhog already ingests Garmin fitness screenshots via a vision LLM — drop an image in a folder, the model reads it, the data lands in DuckDB. Sleep8 follows the exact same pattern. This post walks through adding a new data source, the design decisions along the way, and one embarrassing omission.

---

## The data

Sleep8 tracks three core sleep metrics: resting heart rate, HRV, and breathing rate. Two more — time to fall asleep and deep sleep minutes — appear only on some screens, so they're nullable. The table design reflects that:

```sql
CREATE TABLE IF NOT EXISTS sleep_metrics (
    date DATE PRIMARY KEY,
    resting_hr INTEGER,
    hrv INTEGER,
    breath_rate DECIMAL(4,1),
    time_to_fall_asleep_minutes INTEGER,
    deep_sleep_minutes INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

`date` is the primary key — one row per night, same daily grain as `health_metrics`. `ON CONFLICT DO NOTHING` makes re-running the ingestion script safe.

---

## The architecture decision: separate drop folder

The Garmin ingestion watches `data/drop/`. The first question was whether Sleep8 screenshots could go in the same folder and let the vision LLM figure out which app it's looking at.

It could — the UIs are visually distinct enough. But mixing them means `health.py` needs to know about Sleep8 (and vice versa), and the routing logic has to live somewhere. Separate folders are simpler: each script owns its folder, and neither knows the other exists.

```
data/drop/           ← Garmin screenshots go here
data/drop/sleep8/    ← Sleep8 screenshots go here
```

`config/settings.py` gets one new line:

```python
SLEEP_DROP_FOLDER = BASE_DIR / "data" / "drop" / "sleep8"
```

---

## Getting the date from the filename

The Garmin ingestion asks the vision LLM to extract the date from the screenshot. For Sleep8, a simpler approach: encode the date in the filename and parse it with a regex.

```python
def _date_from_filename(path: Path) -> Optional[str]:
    match = re.search(r"(\d{4}-\d{2}-\d{2})", path.stem)
    return match.group(1) if match else None
```

Any filename containing a `YYYY-MM-DD` pattern works — `sleep8_2026-05-22.png`, `2026-05-22-sleep.png`, whatever. Files without a date are skipped with a warning.

This is more reliable than asking the model to read a date from a mobile UI, and it removes one field from the prompt entirely.

---

## The prompt

With the date handled separately, the prompt is simple:

```python
PROMPT = """You are extracting sleep data from a Sleep8 app screenshot.

Return a JSON object with:
{
  "resting_hr": <integer bpm or null>,
  "hrv": <integer ms or null>,
  "breath_rate": <float breaths/min or null>,
  "time_to_fall_asleep_minutes": <integer or null>,
  "deep_sleep_minutes": <integer or null>
}

Return null for any metric not visible. No explanation, just JSON."""
```

Explicit nulls for missing fields, no date field, no extra instructions. The model returns a JSON blob, a regex extracts it, and the script merges in the filename date before inserting.

---

## The ingestion script

`ingestion/sleep.py` follows the same structure as `ingestion/health.py`:

```
for each image in SLEEP_DROP_FOLDER:
    1. extract date from filename (skip if missing)
    2. send image to vision LLM via Ollama
    3. parse JSON response
    4. upsert into sleep_metrics
    5. move image to sleep8/processed/
```

```python
def run() -> None:
    PROCESSED_DIR.mkdir(parents=True, exist_ok=True)
    images = [p for p in SLEEP_DROP_FOLDER.iterdir() if p.suffix.lower() in IMAGE_EXTS]

    con = duckdb.connect(str(DB_PATH))
    try:
        for image_path in images:
            date = _date_from_filename(image_path)
            if not date:
                print(f"  Skipping: no YYYY-MM-DD date found in filename.")
                continue
            raw = _query_ollama(image_path)
            metrics = _parse_metrics(raw)
            if metrics:
                metrics["date"] = date
                _insert(con, metrics)
                shutil.move(str(image_path), PROCESSED_DIR / image_path.name)
    finally:
        con.close()
```

To run it:

```bash
venv/bin/python ingestion/sleep.py
```

---

## Schema migrations

The `sleep_metrics` table is created fresh in `schema.py`. But the two nullable columns (`time_to_fall_asleep_minutes`, `deep_sleep_minutes`) were added after the initial design, so the schema handles both cases — create and existing table:

```python
con.execute("""
    CREATE TABLE IF NOT EXISTS sleep_metrics (
        date DATE PRIMARY KEY,
        resting_hr INTEGER,
        hrv INTEGER,
        breath_rate DECIMAL(4,1),
        time_to_fall_asleep_minutes INTEGER,
        deep_sleep_minutes INTEGER,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
""")

con.execute("ALTER TABLE sleep_metrics ADD COLUMN IF NOT EXISTS time_to_fall_asleep_minutes INTEGER")
con.execute("ALTER TABLE sleep_metrics ADD COLUMN IF NOT EXISTS deep_sleep_minutes INTEGER")
```

The `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` pattern (same as the `activities` table uses) means re-running `schema.py` is always safe.

---

## The embarrassing part

The folder `data/drop/sleep8/` was never created. The script creates `sleep8/processed/` lazily on first run via `PROCESSED_DIR.mkdir(parents=True, exist_ok=True)` — but if `sleep8/` itself doesn't exist, that call creates both. Except the script iterates over `SLEEP_DROP_FOLDER` before it gets to that line, so it would error trying to list a directory that doesn't exist.

Fixed with:

```bash
mkdir -p data/drop/sleep8
```

Worth noting: the `processed/` subdirectory is still created lazily on first run, which is fine. The parent just needs to exist before the script starts.

---

## What's next

Sleep8 data is now queryable alongside health and activity data. Cross-source questions like "did my HRV improve on days I ran?" are now possible — the agent already has SQL tools that can join across tables.
