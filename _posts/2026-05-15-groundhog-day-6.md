---
layout: post
title: "Day 6 — Persistent Memory + Embeddings"
date: 2026-05-15
categories: groundhog
---

## Persistent Memory + Embeddings

The agent can now remember facts across sessions and recall them semantically.

**How it works:**

Two new tools added to the agent:

- `remember(fact)` — embeds the fact using `nomic-embed-text` and stores it in DuckDB
- `recall(query)` — embeds the query, runs cosine similarity against all stored facts, returns the top matches

The memory table in DuckDB:

```sql
CREATE TABLE memory (
    id VARCHAR PRIMARY KEY,
    fact TEXT,
    embedding FLOAT[],
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

Similarity search is pure SQL using DuckDB's built-in function:

```sql
SELECT fact, list_cosine_similarity(embedding, ?) AS score
FROM memory
ORDER BY score DESC
LIMIT 3
```

**Why DuckDB instead of FAISS:**

FAISS uses optimized index structures that are much faster at scale. But for personal memory — dozens to a few hundred facts — DuckDB's cosine similarity is fast enough and keeps everything in one place with no extra dependency.

**Problem: Agent calling too many tools**

After `remember()` succeeded, the model kept calling `recall`, `get_latest_price`, `get_recent_activities`, and `get_health_summary` — burning through all 5 tool rounds without ever giving a final answer.

Fix: tightened the system prompt to tell the model to stop once it has what it needs, and raised `MAX_TOOL_ROUNDS` to 8:

```
Call only the tools needed to answer the question.
Once you have enough information, stop calling tools and give your final answer.
For simple requests like remembering a fact, call remember() once and immediately confirm to the user.
```

**Problem: Model ignoring recalled memories**

Even after recalling the correct memory, the model answered based on its own training knowledge instead. For example, storing "never refer to Bitcoin as crypto" and then asking "is Bitcoin crypto?" returned a long-winded answer explaining why Bitcoin is technically a cryptocurrency — completely ignoring the stored preference.

The fix was making the system prompt more direct:

```
When recall() returns memories, answer ONLY based on those memories.
Do not add your own knowledge, reasoning, or qualifications. State the answer directly as fact.
```

The key word is ONLY. Without it the model treats recalled memories as one input among many and still reasons from its training data. With it, the memory becomes the authoritative source.

## Concepts touched
- Embeddings and vector similarity search
- RAG foundations — structured SQL data + unstructured memory in the same agent
- DuckDB `FLOAT[]` array type and `list_cosine_similarity()`
- Prompt engineering to constrain agentic behavior
