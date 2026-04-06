---
title: "ADR: OpenRouter Over OpenAI Plus"
tags: [decision, infrastructure, cost]
created: 2026-04-06
updated: 2026-04-06
status: active
sources:
  - markviewer/hermes/archive/2026-04-04-openrouter-vs-openai.md
  - markviewer/hermes/archive/2026-04-04-model-strategy-v2.md
related:
  - model-strategy.md
  - ../projects/hermes.md
  - ../reference/accounts-and-services.md
---

# Decision: OpenRouter Over OpenAI Plus

## Context

Hermes needs access to multiple LLM providers (OpenAI, Google, DeepSeek, Qwen). Calvin was paying £20/month for OpenAI Plus alongside API costs.

## Decision

Cancel OpenAI Plus. Use OpenRouter as the sole LLM API provider. Redirect the £20/month to OpenRouter credits.

## Why

- **Single API key** for 290+ models across all providers — no per-provider auth management
- **No token expiry** — OpenRouter keys work until revoked (OpenAI OAuth tokens rotate every 10 days)
- **Pay-per-use only** — credits never expire, no subscription
- **OpenAI SDK compatible** — change `base_url` to `https://openrouter.ai/api/v1`, everything else works
- **All OpenAI models available** without an OpenAI account (GPT-4o, GPT-5, GPT-5.4, o3, o4)

## Cost Comparison (5M input + 1M output tokens/month, GPT-5.4)

| Route | Cost |
|---|---|
| OpenAI Plus subscription | £20/month (flat, doesn't reduce API costs) |
| OpenAI API direct | ~$27.50/month |
| OpenRouter API | ~$29.01/month (same rates + 5.5% platform fee) |

The Plus subscription is a separate billing system from the API — it doesn't reduce API costs. For headless/agent workflows, the ChatGPT web UI (DALL-E, custom GPTs, browsing) isn't useful.

## What We Lose

ChatGPT web/mobile UI with DALL-E, custom GPTs, web browsing, priority chat. None of these matter for agent-driven workflows.

## Result

Monthly cost dropped from ~£322 (Plus + OpenClaw API) to ~£5.50 (Hermes + GLM Lite). OpenRouter credits from the cancelled subscription cover ~5 months of Hermes operation per £20 top-up.
