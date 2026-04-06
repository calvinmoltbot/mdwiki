---
title: Cron Pipeline
tags: [hermes, infrastructure, automation]
created: 2026-04-06
updated: 2026-04-06
status: active
related:
  - ../projects/hermes.md
  - ../systems/mac-mini.md
---

# Cron Pipeline

Hermes runs automated scheduled tasks via a cron system. Jobs follow a two-stage pipeline pattern and are defined in `~/.hermes/cron/jobs.json`.

## Two-Stage Pattern

Every cron job has the same structure:

**Stage 1 — Pre-run script** (Python/shell, no LLM):
- Fetches or computes raw data (RSS feeds, token stats, file scans)
- Outputs structured text (usually markdown) to stdout
- Scripts live in `~/.hermes/scripts/`

**Stage 2 — LLM synthesis**:
- Hermes injects the script output into a prompt
- The model (default: Gemini 2.5 Flash via OpenRouter) synthesizes a human-readable summary
- Result is delivered to Telegram or kept local

Each run starts a **fresh session** with zero context — no memory of previous runs. This prevents context snowballing, which was the critical lesson from OpenClaw (it burned $3/day before this was fixed).

## Current Jobs

| Job | Schedule | Script | Delivery |
|---|---|---|---|
| YouTube Briefing | Daily 8:00 AM | `youtube_feeds.py` | Telegram |
| RSS/Blog Briefing | Daily 8:05 AM | `newsboat_process.py` | Telegram |
| Interests Briefing | Sun/Wed 6:00 PM | (calvin-interest-databases skill) | Telegram |
| Evening Digest | Daily 6:00 PM | `token_stats.py` | Telegram |
| Log Rotation | Sunday 2:00 AM | (inline shell) | Local |

**Daily token spend:** ~120K tokens/day across all jobs = ~$0.02/day on Gemini 2.5 Flash.

## Model Routing

- **Default:** `google/gemini-2.5-flash` via OpenRouter
- **Fallback:** `deepseek/deepseek-v3.2` via OpenRouter
- **Smart routing:** Simple messages (<160 chars, <28 words) go to `qwen/qwen3.6-plus:free`

## Adding a New Job

1. Write the pre-run script in `~/.hermes/scripts/`
2. Add a job entry to `~/.hermes/cron/jobs.json` with: id, name, prompt, schedule (cron expression), script path, delivery platform
3. The scheduler picks it up automatically

## Key Lessons

- **No context accumulation** — fresh session every run
- **Script does the heavy lifting** — the LLM only synthesizes, never fetches
- **Keep output short** — Telegram messages should be scannable
- **Preserve links** — markdown links in script output must survive LLM synthesis
