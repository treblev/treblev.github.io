---
layout: post
title: "Day 7 — Multi-step Reasoning + Model Upgrade"
date: 2026-05-16
categories: groundhog
---

## Multi-step Reasoning + Model Upgrade

M5 was supposed to be about teaching the agent to plan before acting. It turned into a lesson about model selection.

**What M5 adds:**

Three new functions in `agent/query.py`:

- `_plan(question, schema)` — before the execution loop, asks the model to produce a numbered plan: which tools to call, in what order, and why. No tools are called here — just thinking.
- `_synthesize(question, messages)` — if the agent hits `MAX_TOOL_ROUNDS` without giving a final answer, this fires a final no-tools call with all the accumulated results: "based only on what you found, answer directly."
- The execution loop injects the plan into the system prompt so the model follows it during tool calls.

The architecture is Plan → Execute → Synthesize (if needed). This is the ReAct pattern — Reason then Act — applied to a local agent.

**The qwen2.5-coder:32b problem:**

The first test: "compare my workout frequency this week vs last week."

The model ran the right SQL and got the right data — 6 workouts this week, 0 last week (one row returned, last week absent = zero). Simple question, correct data in hand. But instead of answering, it:

1. Called `remember()` to save the intermediate result to persistent memory
2. Called `recall()` to search memory for "weekly_workout_frequencies"
3. Re-ran the same SQL three more times with minor variations
4. Tried a UNION query (with a syntax error)
5. Hit `MAX_TOOL_ROUNDS` and returned "Agent reached max tool rounds without a final answer."

Five rounds of tool calls for a question that needed one.

The fixes tried, in order:

```
# attempt 1
"Call only the tools needed. Once you have enough information, stop."

# attempt 2
"Do NOT call remember() for intermediate results — only when the user explicitly asks."

# attempt 3
"NEVER call remember() unless the user's message explicitly uses the word 'remember' or 'save'."

# attempt 4 — updated the recall() tool description
"Search persistent memory for personal opinions, preferences, and stated beliefs.
Do NOT use for factual data questions."

# attempt 5
"For comparison queries: if only one period appears in results, the missing period is zero.
State the answer directly."
```

Each fix addressed the symptom of one run but the model found a new way to go wrong on the next. After five prompt iterations the agent was still looping. The model had hit its ceiling.

**The real fix: switch models.**

`qwen3:32b` is Qwen's latest generation — better instruction-following, built-in chain-of-thought reasoning. Swapped `OLLAMA_SQL_MODEL` in `config/settings.py` and ran the same question.

```
[plan] 1. run_sql: count activities this week
       2. run_sql: count activities last week

[tool] run_sql(...) → count_star(): 6
[tool] run_sql(...) → count_star(): 0

Agent: You had 6 workouts this week compared to 0 last week.
       Your workout frequency increased by 6 sessions week-over-week.
```

Two tool calls. Correct answer. Done.

**The lesson:**

Prompt engineering has a ceiling that varies by model. With a weak model you can spend hours chasing symptoms — tightening instructions, adding guardrails, handling edge cases — and still get unreliable behavior. The same architecture that took five prompt iterations to half-work on qwen2.5-coder worked first try on qwen3.

Time spent on model selection is not wasted. It's often the highest-leverage decision in the stack.

## Concepts touched
- ReAct pattern (Reason + Act): planning step before tool execution
- Synthesis fallback: forced no-tools answer when the agent loops
- Model selection as a first-class engineering decision, not an afterthought
- Prompt engineering ceiling — when to stop tuning and swap the model
