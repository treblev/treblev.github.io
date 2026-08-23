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

The questions exercised a mixture of structured lookups, empty-result handling,
filtered listings, and semantic retrieval. This post intentionally reports only
aggregate results; it does not include the questions, responses, or underlying
personal records.

> **Baseline status:** The latency is higher than desired. This post will be
> edited after latency improvements are implemented and the same evaluation is
> rerun.

## Accuracy and successful-response latency

The latency statistics in this first table include successful responses only.
Standard deviation is the population standard deviation across the completed
responses in this run.

| Model | Factual passes | Responses returned | Timeouts | Pass rate | Average | Median | Standard deviation | Minimum | Maximum |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Qwen 3.6 | 15/15 | 15/15 | 0 | 100.00% | 48.35s | 44.42s | 14.72s | 30.86s | 79.29s |
| Gemma 4 | 14/15 | 14/15 | 1 | 93.33% | 41.27s | 34.09s | 18.84s | 22.78s | 90.19s |

Qwen completed every question correctly. Gemma returned correct answers for all
14 questions it completed, but one request did not return an answer and was
terminated after more than 686.94 seconds.

Gemma was faster on the median successful request, but successful-response
statistics alone hide the operational cost of that timeout. Treating the last
measured timeout duration as a lower bound produces the following attempt-level
view:

| Model | Average | Median | Standard deviation | Minimum | Maximum |
|---|---:|---:|---:|---:|---:|
| Qwen 3.6 | 48.35s | 44.42s | 14.72s | 30.86s | 79.29s |
| Gemma 4 | at least 84.32s | 35.40s | at least 162.08s | 22.78s | more than 686.94s |

The distinction between median and average matters here. Gemma's median shows
that a typical successful response was relatively quick, while its timeout
caused much worse tail latency and reliability. Qwen was slower on the typical
successful response but considerably more consistent across the complete test.

## Latency is now the main problem

Accuracy was strong enough to move the focus to responsiveness. Waiting roughly
half a minute to more than a minute for an ordinary answer is not the experience
Groundhog should provide, and an unbounded request lasting many minutes is not
acceptable.

The next work will reduce unnecessary model inference, add faster paths for
structured questions, and enforce a practical timeout. After those improvements
are in place, the same 15-question evaluation will be run again. This post will
then be edited with the new latency statistics so the improvement can be
measured against this baseline rather than judged informally.
