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

> **Baseline status:** The latency is higher than desired. This post will be
> edited after latency improvements are implemented and the same evaluation is
> rerun.

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

Request traces showed that approximately 92–94% of total response time was
spent in language-model inference. Database access, tool execution, and
embedding retrieval were comparatively small parts of the total. Two factors
account for most of the delay:

1. **Each question invokes the model repeatedly.** The requests are treated very non-deterministically and so the LLM is checking the entire DB tables list one by one before deciding on its action. 

2. **Each model pass receives more context than it needs.** Even straightforward
   questions have very broad instructions about every data type we have started to collect over the iterations. BLOAT! 
   Improvement here would be to make the action be more deterministic by reconfiguring the way we treat the requests. Codex indicates a routing schema is best. 

## Latency is now the main problem

Accuracy is not the problem now. Responsiveness is. Waiting 40 seconds for each query is problematic. 

