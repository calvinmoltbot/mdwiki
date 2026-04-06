---
title: Hermes Agent
tags: [hermes, agent, infrastructure]
created: 2026-04-06
updated: 2026-04-06
status: active
related:
  - ../systems/cron-pipeline.md
  - ../systems/mac-mini.md
  - ../systems/claude-code.md
  - ../reference/accounts-and-services.md
---

# Hermes Agent

Hermes is a full AI agent system running on the [Mac Mini](../systems/mac-mini.md). It has two modes: terminal CLI (`hermes`) and messaging gateway (`hermes gateway`). The codebase lives at `~/.hermes/hermes-agent/`.

## What Hermes Does

- **Messaging gateway** — receives and responds to messages from Telegram (primary), with support for Discord, Slack, and others
- **Cron automation** — runs scheduled tasks via the [cron pipeline](../systems/cron-pipeline.md) (daily briefings, log rotation)
- **Agent handoffs** — coordinates with Claude Code via `calvinmoltbot/handoffs` repo

## Architecture

### Gateway
The gateway is the always-running process that handles messaging:
- `gateway/run.py` — main loop, slash commands, message dispatch
- `gateway/session.py` — conversation persistence
- `gateway/platforms/telegram.py` — Telegram adapter (canonical implementation)
- State tracked in `~/.hermes/gateway_state.json`
- PID file: `~/.hermes/gateway.pid`

### Agent Core
- `run_agent.py` — AIAgent class, conversation loop
- `agent/prompt_builder.py` — system prompt assembly
- `agent/context_compressor.py` — auto compression (0.5 threshold, 0.2 target ratio)
- `agent/skill_commands.py` — skill slash commands

### CLI
- `hermes_cli/main.py` — entry point
- `hermes_cli/commands.py` — central slash command registry (`COMMAND_REGISTRY`)
- `hermes_cli/model_switch.py` — model switching

## Model Configuration

| Role | Model | Provider |
|---|---|---|
| Default | `google/gemini-2.5-flash` | OpenRouter |
| Fallback | `deepseek/deepseek-v3.2` | OpenRouter |
| Simple messages | `qwen/qwen3.6-plus:free` | OpenRouter |

Smart routing sends simple turns (<160 chars, <28 words) to the free model.

## Telegram Integration

Primary messaging platform. Uses `python-telegram-bot` library.
- Supports: text, media, voice memos (transcribed via local Whisper), inline keyboards
- Formatting: MarkdownV2 with fallback to plain text
- Cron job results delivered here
- Chat ID: `8262919418`

## Discovery & Curation Philosophy

Hermes is Calvin's personal intelligence agent — finds signal in noise. The cron pipeline feeds The Bridge (a personal feed dashboard).

**Quality rules:**
- **3-5 genuinely interesting items** beats 15 mediocre links
- **Summary, link, why it matters** — no raw dumps
- **If nothing changed, say so** — "nothing newsworthy" beats filler
- **Interest separation**: Home (AI/LLM news, coding, podcasts, audiobooks) vs Work (Copilot, enterprise AI, governance) kept in separate streams

**What not to repeat from OpenClaw:**
- Too-frequent sweeps (noise + context bloat) → fewer, higher-quality sweeps
- Accumulated context between runs → fresh context each run
- Complex multi-agent orchestration → single agent, single responsibility per run
- Automation before reading experience → reading experience is priority #1

See [agent design lessons](../patterns/agent-design-lessons.md) for the full cost/design analysis.

## Session Management

- Sessions reset after 1440 idle minutes (24h) or at 4 AM daily
- Compression enabled — protects last 20 conversation snapshots
- Memory: 2200 char limit for user memory, 1375 for user profile

## Skills

Skills live in `~/.hermes/skills/` and can be enabled/disabled per platform (CLI, Telegram, Discord). The gateway generates Telegram bot menu commands from the skill registry.

## Configuration

- Config: `~/.hermes/config.yaml` (version 11)
- Secrets: `~/.hermes/.env`
- Logs: `~/.hermes/logs/` (rotated weekly when >10MB)
- Default personality: `kawaii`
