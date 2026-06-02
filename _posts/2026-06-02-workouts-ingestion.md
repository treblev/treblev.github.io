---
layout: post
title: "Ingesting SugarWOD Workouts via Vision LLM"
date: 2026-06-02
categories: groundhog
---

Groundhog can already parse Garmin screenshots for health metrics and Sleep8 screenshots for sleep data. The same vision LLM pipeline now handles SugarWOD — extracting daily workout cards into a `workouts` table so the agent can answer questions like "what did I do on Tuesday?" or "find me an AMRAP from last week."

---

## The data

SugarWOD shows workout cards — each card has a name, a category (Fitness, Performance, HYROX), a structure type (AMRAP, EMOM, rotating intervals, strength), and a full description with exercises and reps. The table design:

```sql
CREATE TABLE IF NOT EXISTS workouts (
    id VARCHAR PRIMARY KEY,
    date DATE,
    day_of_week VARCHAR,
    name VARCHAR,
    category VARCHAR,
    structure_type VARCHAR,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

`id` is a SHA-256 hash of `date + name + description[:50]`, so re-running the ingestion is safe.

---

## Weekly view vs daily view

The first attempt used a weekly calendar screenshot (Mon–Sun, all workout cards visible at once). The vision model was asked to extract every card across all seven days in one shot.

It timed out after 14 minutes.

A dense weekly calendar with 15+ workout cards is too much for `qwen3-vl` to read, serialize, and return as structured JSON in one pass. Switched to daily screenshots — one day at a time, much faster.

---

## Prompt design

The prompt always returns a flat JSON array, one element per workout card, regardless of whether the screenshot shows a single section or multiple:

```python
PROMPT = """You are extracting workout data from a SugarWOD screenshot.

Always return a JSON array. Each element represents one workout card.

For each workout card, return:
{
  "day_of_week": <"MON"|"TUE"|...|"SUN" or null>,
  "date_day": <integer day-of-month or null>,
  "date": <"YYYY-MM-DD" if visible, otherwise null>,
  "name": <card header text>,
  "category": <"Fitness"|"Performance"|"HYROX"|...>,
  "structure_type": <"amrap"|"emom"|"rotating"|"for_time"|"strength"|"intervals"|null>,
  "description": <full workout text>
}
..."""
```

`structure_type` is inferred from keywords in the text rather than asked for explicitly — the model applies the rules rather than guessing:

```
"amrap"     → text contains "AMRAP"
"emom"      → text contains "EMOM"
"rotating"  → text contains "rotating"
"for_time"  → text contains "for time" or a time cap like "(15 cap)"
"strength"  → a lift with a rep scheme like "8-8-6-6-4-4"
"intervals" → timed work/rest blocks like "30s work / 2 min rest"
```

---

## Date fallback

SugarWOD daily screenshots don't always show the full date — sometimes just the day of week, sometimes nothing. Rather than asking the model to infer it, the ingestion script falls back to today's date:

```python
def _fill_missing_date(workouts: list[dict]) -> None:
    today = date.today()
    for w in workouts:
        if not w.get("date"):
            w["date"] = today.isoformat()
        if not w.get("date_day"):
            w["date_day"] = today.day
        if not w.get("day_of_week"):
            w["day_of_week"] = today.strftime("%a").upper()
```

The assumption is: you drop the screenshot the same day you take it. For retrospective ingestion, rename the file to include `YYYY-MM-DD` and extend the script to parse it from the filename — the same approach used for Sleep8.

---

## Notebook first

The prompt and parsing logic were developed in `notebooks/workouts_vision_prompt_evals.ipynb` before being moved into the ingestion script. The notebook has three cells after the prompt:

1. **Ollama call** — encodes the image, sends it, prints the raw response
2. **Parser** — extracts the JSON array, applies the date fallback, prints a summary line per workout

Once the output looked right, the functions were copied directly into `ingestion/workouts.py` with minimal changes (underscored names, error handling, `shutil.move` to processed folder).

---

## The macOS filename gotcha

macOS screenshot filenames use a **narrow no-break space** (` `) before "AM"/"PM" — not a regular space. So `"Screenshot 2026-06-02 at 6.12.17 AM.png"` in Finder is actually `"Screenshot 2026-06-02 at 6.12.17 AM.png"` on disk.

Hardcoding the path with a regular space causes `FileNotFoundError`. The fix is to copy the path programmatically:

```python
import os
files = os.listdir('/path/to/folder')
print(repr(files[0]))  # reveals the  
```

Then paste the repr'd string (with the actual unicode character) into the notebook.

---

## Running it

Drop a SugarWOD screenshot into `data/drop/workouts/` and run:

```bash
venv/bin/python ingestion/workouts.py
```

Processed images move to `data/drop/workouts/processed/`. The result for today's screenshot:

```
Processing Screenshot 2026-06-02 at 6.12.17 AM.png...
  Inserted: TUE | Murph Recovery | rotating
```

The `workouts` table is now queryable through the agent alongside health, sleep, and activity data.
