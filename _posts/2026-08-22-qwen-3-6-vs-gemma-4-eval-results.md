---
layout: post
title: "Qwen 3.6 vs Gemma 4 Eval Results"
date: 2026-08-22
categories: groundhog
---

> **Test environment:** These tests were conducted on my local Mac, where both
> Qwen 3.6 and Gemma 4 were running locally.

This end-to-end model evaluation compares two local answer models using the
same set of 15 questions. The database, tools, prompts, and embedding-based
retriever were held constant so that only the answer model changed.

Questions are related to personal data so we ommited for this post.

> **Baseline status:** The baseline was too slow. The same evaluation was rerun
> after two targeted latency changes.

## Accuracy and successful-response latency

The latency statistics in this first table include successful responses only. SD applies for completed questions only.

| Model | Factual passes | Responses returned | Timeouts | Pass rate | Average | Median | Standard deviation | Minimum | Maximum |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Qwen 3.6 | 15/15 | 15/15 | 0 | 100.00% | 48.35s | 44.42s | 14.72s | 30.86s | 79.29s |
| Gemma 4 | 14/15 | 14/15 | 1 | 93.33% | 41.27s | 34.09s | 18.84s | 22.78s | 90.19s |

Qwen completed every question correctly but slower than Gemma.
Gemma was faster on the median successful request, but also had one failed answer:

| Model | Average | Median | Standard deviation | Minimum | Maximum |
|---|---:|---:|---:|---:|---:|
| Qwen 3.6 | 48.35s | 44.42s | 14.72s | 30.86s | 79.29s |
| Gemma 4 | at least 84.32s | 35.40s | at least 162.08s | 22.78s | more than 686.94s |

Overall Gemma 4 seems an improvement over Qwen but this is not the 3.8 version. So it will be interesting to see how the results look for that comparison.

## Two main causes of slowness

Request traces showed that 92–94% of response time was model inference. Two
things caused most of the drag:

1. **Repeated model passes:** The agent repeatedly inspected tools and schema
   before answering.
2. **Bloated context:** Every pass carried instructions and data definitions
   unrelated to the question.

## Improvements made for latency improvement

1. **Deterministic routing:** Parse each request, call only the relevant tool,
   and keep broad schema/tool context out of confident routes.
2. **One verifier:** Combine grounding and internal-detail checks into one review
   call instead of two sequential calls.

## Post-change results

The routed path cut successful-response latency and eliminated Gemma's timeout:

| Model | Legacy factual | Legacy timeouts | Legacy average | Routed factual | Routed timeouts | Routed average | Improvement |
|---|---:|---:|---:|---:|---:|---:|---:|
| Qwen 3.6 | 15/15 | 0 | 48.35s | 15/15 | 0 | 9.42s | 80.5% faster |
| Gemma 4 | 14/15 | 1 | 41.27s* | 15/15 | 0 | 6.38s | 84.5% faster |

\* Gemma's legacy average covers successful responses only. Its timed-out request
ran for more than 686.94 seconds; including that lower bound raises the attempt
average to at least 84.32 seconds.
