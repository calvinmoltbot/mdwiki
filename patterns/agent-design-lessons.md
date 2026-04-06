---
title: Agent Design Lessons
tags: [patterns, agents, cost, hermes]
created: 2026-04-06
updated: 2026-04-06
status: active
sources:
  - markviewer/hermes/archive/2026-04-04-openclaw-activity-analysis.md
related:
  - ../decisions/model-strategy.md
  - ../systems/cron-pipeline.md
  - ../projects/hermes.md
---

# Agent Design Lessons

Five hard-learned lessons from OpenClaw (Calvin's previous agent system that burned $3/day before being retired). These apply to any LLM-powered agent.

## 1. Heartbeats Must Be Lightweight

Sending full context (50-90K tokens) just to check "anything need doing?" is catastrophically expensive. A heartbeat should be ~500 tokens: minimal system prompt + current time. Read memory only if the heartbeat detects something worth acting on.

## 2. Context Must Be Managed, Not Accumulated

OpenClaw appended every heartbeat response to the conversation, causing exponential growth: 50K → 93K tokens per cycle. **Every scheduled run must start with a fresh, minimal context.** This is the single most important rule. The [cron pipeline](../systems/cron-pipeline.md) enforces this by design.

## 3. Use the Cheapest Model for Routine Tasks

68% of OpenClaw's API calls were pure tool routing — the model just picked which tool to call. A free or near-free model handles this fine. Reserve expensive models for actual reasoning. See [model strategy](../decisions/model-strategy.md).

## 4. The 188:1 Ratio Is a Red Flag

OpenClaw's token ratio was 188:1 input:output — 64M prompt tokens producing 340K completion tokens over 7 days. If your agent's input-to-output ratio is wildly skewed, you're paying to re-read context, not to think. Fix the context, not the model.

## 5. Monitor on Day One

Check the OpenRouter dashboard after the first day. Set budget alerts. Review the first few cron job logs for sensible token counts. The difference between $0.05/week and $20/week is usually one misconfigured context window.

## The OpenClaw Numbers

| Metric | Value |
|---|---|
| Duration | 7 days |
| Total API calls | 1,411 |
| Prompt tokens | 64 million |
| Completion tokens | 340K |
| Total spend | $20.96 |
| Heartbeat cost alone | ~$8.32/week |
| Hermes equivalent | ~$0.05/week |

The 400x cost reduction came from applying these five rules, not from switching models.
