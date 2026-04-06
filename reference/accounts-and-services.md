---
title: Accounts and Services
tags: [reference, infrastructure]
created: 2026-04-06
updated: 2026-04-06
status: active
related:
  - ../systems/mac-mini.md
  - ../projects/hermes.md
---

# Accounts and Services

## GitHub

| Field | Value |
|---|---|
| Primary account | **calvinmoltbot** |
| Old account | calvinorr (migrating repos from here) |
| Vercel GitHub App | All-repo access on calvinmoltbot |
| Standard labels | `bug`, `design`, `ready-for-claude`, `for-hermes` |
| Handoff repo | `calvinmoltbot/handoffs` |

## Vercel

| Field | Value |
|---|---|
| Team | **calvin-orrs-projects** |
| Custom domain | **warmwetcircles.com** |
| CLI tools | `vercel` / `vc` |
| Deploy trigger | Push to `main` → auto-deploy via GitHub integration |

Always use subdomains (`garden.warmwetcircles.com`) not random `.vercel.app` URLs. Use the Vercel CLI for operations — don't use browser automation.

## Tailscale

| Field | Value |
|---|---|
| Mac Mini IP | **100.90.11.37** |
| Purpose | Network bridge between MacBook and Mac Mini |

## OpenRouter

| Field | Value |
|---|---|
| Purpose | LLM API provider for Hermes |
| Base URL | `https://openrouter.ai/api/v1` |
| Default model | `google/gemini-2.5-flash` |
| Fallback | `deepseek/deepseek-v3.2` |
| Free tier | `qwen/qwen3.6-plus:free` (for simple messages) |

## Key Conventions

- **Env vars with Vercel CLI:** Use `printf` not `echo` when piping values to `vercel env add` — echo appends a trailing `\n`
- **GitHub labels:** Must be created before use with `gh label create --force`
- **Domain pattern:** `<app>.warmwetcircles.com` for all deployed projects
