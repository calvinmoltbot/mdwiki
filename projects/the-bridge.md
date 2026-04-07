---
title: The Bridge
tags: [the-bridge, dashboard, hermes, next.js]
created: 2026-04-07
updated: 2026-04-07
sources:
  - session 2026-04-07 — documented separation of concerns and data contract
status: active
related:
  - ../projects/hermes.md
  - ../systems/cron-pipeline.md
  - ../patterns/stitch-design-workflow.md
---

# The Bridge

Personal intelligence feed dashboard powered by Hermes Agent. Reads cron output files directly from the Hermes filesystem — no database of its own.

**Repo:** `calvinmoltbot/the-bridge`
**URL:** `http://100.90.11.37:3020` (Tailscale only, no auth)
**Stack:** Next.js 16, React 19, Tailwind CSS v4, TypeScript

## Separation of Concerns

The Bridge is **read-only** — it displays what Hermes produces but never drives Hermes behavior. These are two separate repos, worked on in separate Claude sessions:

| Concern | Repo | Session focus |
|---|---|---|
| **What the agent does** (cron jobs, skills, prompts, feeds) | `hermes-agent` (~/.hermes/) | Hermes session |
| **How Calvin reviews the output** (dashboard UI, layout, design) | `the-bridge` | This session |

If a Bridge feature implies Hermes needs to change (e.g. new data format, new job type), that's a **handoff** — note it as an issue, don't implement it here.

## Data Contract

Bridge reads Hermes output via filesystem. No API, no database of its own (except read-only SQLite).

| Bridge reads | Hermes writes | Format |
|---|---|---|
| `~/.hermes/cron/output/{jobId}/{datetime}.md` | Cron job results | Markdown with `**Run Time:**` header, `## Response` / `## Error` sections |
| `~/.hermes/cron/jobs.json` | Job definitions | JSON: `{ jobs: [{ id, name, schedule, state, last_run_at, ... }] }` |
| `~/.hermes/gateway_state.json` | Gateway status | JSON: `{ pid, gateway_state, platforms: { telegram: { state } } }` |
| `~/.hermes/state.db` | Token usage | SQLite: `sessions` table (started_at, estimated_cost_usd, input/output_tokens) |

## Architecture

- **Server Components** read Hermes data directly via `fs` (cron output, jobs.json, gateway_state.json)
- **better-sqlite3** reads `~/.hermes/state.db` for token usage (read-only)
- **3 API routes** (`/api/gateway`, `/api/cron`, `/api/tokens`) for client-side polling
- **One data module** (`src/lib/hermes.ts`) encapsulates all filesystem access
- **YouTube embeds** via `src/lib/youtube-embed.ts` — converts YouTube watch URLs to inline iframes

## Pages

| Route | Purpose | Data source |
|---|---|---|
| `/` (Today) | Tabbed feed: YouTube, RSS, Interests, Digest | Cron output files |
| `/history` | Browse past briefings by date | Cron output files |
| `/cron` | Job status, schedules, errors | jobs.json |
| `/usage` | Token stats, gateway health, model info | state.db, gateway_state.json |

## Design System

Uses Stitch-generated "The Quiet Authority" design system:
- **Fonts:** Manrope (headlines), Inter (body), JetBrains Mono (technical data)
- **Palette:** warm charcoal base (#0e0e0f), #e6e5e8 primary text, #f6ffc0 lime accent
- **Surfaces:** tonal layering (no borders), glass panels with 16px blur
- **Icons:** Material Symbols Outlined
- Stitch reference HTML files in `stitch-designs/`

See [Stitch design workflow](../patterns/stitch-design-workflow.md) for how this was applied.

## Key gotcha: allowedDevOrigins

Next.js 16 blocks cross-origin WebSocket by default. Since Calvin accesses via Tailscale (100.90.11.37), this breaks React hydration silently — components render server-side but are non-interactive. Fix:

```ts
// next.config.ts
allowedDevOrigins: ["100.90.11.37", "192.168.7.30"],
```

This applies to ANY Next.js 16+ project accessed over the network.

## Dev server

```bash
cd /Users/admin/Dev/Projects/the-bridge
npm run dev  # binds 0.0.0.0:3020 with --webpack (not Turbopack)
```
