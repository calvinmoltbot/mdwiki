---
title: "ADR: Model Strategy"
tags: [decision, hermes, cost, models]
created: 2026-04-06
updated: 2026-04-06
status: active
sources:
  - markviewer/hermes/archive/2026-04-04-model-strategy-v2.md
  - markviewer/hermes/archive/2026-04-04-model-comparison.md
related:
  - openrouter-over-openai.md
  - ../projects/hermes.md
  - ../patterns/agent-design-lessons.md
---

# Decision: Model Strategy — The Cost Ladder

## Context

Agents make many LLM calls. Most are simple (tool routing, heartbeats, status checks). A few need real reasoning. Using the same expensive model for everything wastes money — OpenClaw proved this at $3/day.

## Decision

A tiered model strategy via [OpenRouter](openrouter-over-openai.md):

| Tier | Model | Input $/1M | Output $/1M | Use Case |
|---|---|---|---|---|
| Free | Qwen 3.6 Plus | $0.00 | $0.00 | Simple routing, heartbeats, <160 chars |
| Daily driver | Gemini 2.5 Flash | $0.13 | $0.40 | Conversations, cron synthesis, general tasks |
| Fallback | DeepSeek V3.2 | $0.26 | $0.38 | When Gemini is down or needs better reasoning |
| Premium | GPT-5.4 | $2.50 | $15.00 | Complex analysis, on-demand only |

## Why This Stack

- **Gemini 2.5 Flash** — best price-to-quality for agentic work, native tool calling, 256K context
- **Smart routing** — simple turns (<160 chars, <28 words) auto-route to Qwen free tier. 68% of agent calls are pure tool routing — don't pay for them
- **DeepSeek as fallback** — similar cost, slightly better reasoning, useful when Gemini has outages
- **GPT-5.4 on-demand** — available via OpenRouter pay-per-use, no subscription needed. Switch mid-conversation when quality matters

## Cost Projection

~£5.50/month total (down from £322/month with OpenClaw + OpenAI Plus).

## Future Option

Gemini 2.5 Flash could run locally on the Mac Mini (32GB RAM, Q4 quantization) for zero API cost. Not urgent at current spend levels.
