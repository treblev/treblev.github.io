---
layout: post
title: "What Happened When I Asked Groundhog About MSFT"
date: 2026-08-15
categories: groundhog
---

I asked Groundhog a simple question in Telegram:

```text
/ask What have I written about MSFT recently?
```

The reply was:

> You haven't written any recent notes about MSFT. There are no active stock
> notes for Microsoft currently saved in your records.

That answer was short, but the path to it involved command routing, SQL,
embeddings, semantic search, and a local language model. Here is what actually
happened.

## The command entered Groundhog directly

The `/ask` command was handled by OpenClaw's deterministic ask router. The
router passed the question to Groundhog's local agent instead of asking the
outer chat model to decide what to do.

Groundhog opened a local MCP session and inspected the database schema. It also
looked up the tickers that currently had active user-authored stock notes. Since
the question named `MSFT`, the agent recognized it as a stock-note question.

## The stock-note search used an embedding

Groundhog called its semantic-search tool for the stock-note domain. The
effective search looked like this:

```json
{
  "query": "MSFT",
  "ticker": "MSFT",
  "top_k": 10
}
```

Ollama then generated an embedding using the local model:

```text
qwen3-embedding:0.6b
```

One subtle detail is that the embedding input was `MSFT`, not the entire
sentence `What have I written about MSFT recently?`. The ticker filter narrowed
the search to Microsoft notes, while the vector represented the search term.

Groundhog compared that vector with the stored stock-note vectors using cosine
similarity. The search returned no active MSFT notes.

## SQL provided the final confirmation

Semantic search is useful for finding relevant evidence, but it is not the
source of truth for whether a note exists. Groundhog followed the empty search
with a direct database query against active stock notes:

```sql
SELECT id, ticker, note
FROM stock_notes
WHERE ticker = 'MSFT'
  AND is_deleted = false
ORDER BY id DESC
LIMIT 10;
```

That query also returned no rows. The result was now supported by both the
semantic search and the structured database check.

## Qwen 3.6 wrote the response

The chat model, `qwen3.6:latest`, received the search result and SQL result. It
turned them into the concise Telegram response saying that no active MSFT
notes were saved.

The important separation is:

```text
embeddings → find potentially relevant notes
SQL        → confirm the structured record state
Qwen 3.6   → explain the evidence to the user
```

The embedding model did not answer the question by itself. It helped Groundhog
search the right part of the database. SQL confirmed the absence of an active
record, and the language model communicated that conclusion.

## Why this flow matters

This is the kind of small, inspectable pipeline I want from a personal-data
assistant. Each layer has a narrow job:

- OpenClaw routes the command.
- Groundhog chooses the local tools.
- Embeddings retrieve semantically relevant evidence.
- DuckDB verifies facts.
- Qwen 3.6 produces the final wording.

When the answer is wrong, that separation makes the failure easier to locate.
Was the command routed incorrectly? Did retrieval miss a note? Did SQL return
the wrong rows? Or did the language model misstate the evidence? In this case,
the successful MSFT request made it all the way through the complete pipeline,
and the empty answer was confirmed by the database.
