---
title: The Bridge
tags: [the-bridge, dashboard, hermes, next.js]
created: 2026-04-07
updated: 2026-04-15
sources:
  - session 2026-04-07 — documented separation of concerns and data contract
  - session 2026-04-14 — added /hud (live dashboard), telemetry API, WebSocket integration
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
| `/hud` | Live HUD: gateway, today's spend, system health, cron history, recent sessions, ⌘K palette, chat with Hermes | telemetry API + WebSocket |

## Live HUD (`/hud`)

Heads-up display showing all critical Hermes state at once, with live updates via WebSocket. Patterns adapted from [joeynyc/hermes-hudui](https://github.com/joeynyc/hermes-hudui) but scoped to our data shapes and Quiet Authority design.

### Telemetry API

| Endpoint | Returns | Source |
|---|---|---|
| `GET /api/telemetry/gateway` | Gateway pid + state, Telegram state | gateway_state.json |
| `GET /api/telemetry/cron` | All cron jobs with schedule + last/next runs | jobs.json |
| `GET /api/telemetry/cron/history?limit=N` | Last N runs grouped by jobId | sessions table (id pattern `cron_<jobId>_<ts>`) |
| `GET /api/telemetry/usage` | Today + total cost/tokens/sessions | state.db |
| `GET /api/telemetry/feed` | Recent cron output | filesystem |
| `GET /api/telemetry/memory` | Hermes memory entries | `~/.hermes/memory` |
| `GET /api/telemetry/health` | Mac mini CPU/mem/disk/uptime | node:os + df |
| `GET /api/telemetry/chat/sessions` | Recent chat sessions | state.db |
| `POST /api/telemetry/signal` | Push `{type, paths?, data_types?}` to all WS clients | — |
| `POST /api/hud/chat` | Shells `hermes chat -Q -q` with `--continue <name>` | hermes CLI |

### WebSocket integration

Custom Next.js server (`server.ts` via `npm run dev:ws`) mounts a WebSocket upgrade handler at `/ws` on the same HTTP port (3020). Frontend `useHudWs` hook subscribes, receives `{type: 'data_changed', data_types?: [...], paths?: [...]}` messages, and calls SWR `mutate(key)` to revalidate matching API keys. Exponential backoff reconnect, 30s heartbeat ping.

To trigger a HUD refresh from anywhere (e.g. a Hermes script that just updated state):

```bash
curl -X POST http://localhost:3020/api/telemetry/signal \
  -H 'Content-Type: application/json' \
  -d '{"type":"data_changed","data_types":["cron","sessions"]}'
```

### Critical gotchas

**1. Module-context singleton bug.** Next.js App Router API routes load in a separate module context from the custom server. A naive singleton instantiation will give the route its own `WebSocketManager` instance with zero clients, while the server holds the real connections — broadcasts silently fail. Fix: pin on `globalThis`:

```ts
declare global { var __bridgeWebsocketManager: WebSocketManager | undefined; }
export const websocketManager = globalThis.__bridgeWebsocketManager ?? new WebSocketManager();
if (!globalThis.__bridgeWebsocketManager) globalThis.__bridgeWebsocketManager = websocketManager;
```

**2. Upgrade handler must NOT destroy non-/ws sockets.** Calling `socket.destroy()` on the else branch of the `'upgrade'` listener kills Next.js's HMR WebSocket at `/_next/webpack-hmr`, which silently breaks client hydration — the page renders server-side but never becomes interactive. Only claim `/ws`; let other upgrades fall through.

**3. Chat is non-streaming for v1.** `POST /api/hud/chat` shells out to `hermes chat -Q -q "<msg>" --continue <sessionName>` and waits up to 110s for the full response. Streaming/tool-call display is a future build.

### Cron history derivation

Hermes doesn't expose run history directly, but every cron run creates a session with id `cron_<jobId>_<YYYYMMDD>_<HHMMSS>`. We regex-extract `jobId` from `sessions.id` and group by it. **Fragile** — would break if Hermes changes session naming.

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

Two scripts depending on whether you need the HUD's WebSocket:

```bash
cd /Users/admin/Dev/Projects/the-bridge
npm run dev      # plain Next dev (no WS) — sufficient for /, /history, /cron, /usage
npm run dev:ws   # custom server with WebSocket on /ws — required for /hud
```

Both bind `0.0.0.0:3020` with `--webpack` (not Turbopack — non-localhost connections hang).
