---
title: mytodo — Capture-first todo app
created: 2026-04-29
updated: 2026-05-03
tags: [todo, telegram, voice, ai-gateway, gardening, calvin]
---

# mytodo

Calvin's personal todo app, built around capture-first workflow. Pain point: thinks of tasks faster than he can type, especially while gardening — by the time he's back at a device he's forgotten the original ones.

## Architecture

| Layer | Choice |
|---|---|
| Repo | `calvinmoltbot/mytodo` |
| Domain | `todo.warmwetcircles.com` |
| Frontend | Next.js 16.2.4 (App Router, **webpack mode — not Turbopack**), React 19 |
| Styling | Tailwind v4 + hand-rolled components (no shadcn init — pulled cmdk + lucide-react directly) |
| DB | Neon Postgres via Vercel Marketplace, Drizzle ORM |
| Auth | Clerk (single-user, gated by `APP_OWNER_EMAILS` env var) |
| Voice | Groq Whisper (`whisper-large-v3-turbo`) — free tier, ~10x faster than OpenAI |
| LLM cleanup | Gemini 2.0 Flash Lite via Vercel AI Gateway (auth via `VERCEL_OIDC_TOKEN`) |
| File storage | Vercel Blob (currently public-but-unguessable URLs; #24 to switch to signed) |
| Telegram bot | `@jesteretodo_bot` (separate from Hermes), webhook-driven |

## Capture flow

1. Capture surface (Telegram text/voice/photo, web text/voice) → `prepareCapture(rawInput, source, opts)`
2. **Routing precedence** (in order):
   1. Explicit `name:` prefix in user text
   2. LLM-suggested list (Gemini Flash Lite picks from existing lists)
   3. Per-chat default list (Telegram-only — set via `/list <name>`, persisted in `chat_settings` table)
   4. Global Inbox
3. Insert task with optional `cleanedTitle`/`suggestedTags`/`dueAt`/`recurrence`
4. UI surfaces non-list suggestions as ✨ chips (cleanedTitle, tags) for accept/dismiss

For voice sources, the cleaned title is applied directly (raw transcription preserved in `transcription` column). For text the user's verbatim input is kept and chips are shown.

## Why some choices

- **Groq over OpenAI Whisper** — Calvin had no OpenAI key. Groq's free tier covers personal use; OpenAI-compatible API; `whisper-large-v3-turbo` is faster.
- **AI Gateway over direct Gemini** — `VERCEL_OIDC_TOKEN` auto-provisioned on Vercel deploys, no separate API key. Gateway also gives cost dashboards and easy provider swapping. Note: Gateway exposes `languageModel` + `imageModel`, not `transcriptionModel` — that's why Whisper is direct to Groq.
- **Anthropic models rejected** — too expensive for a single-user personal app per Calvin.
- **Webpack not Turbopack** — Calvin's Tailscale + non-localhost connection setup hangs with Turbopack dev. Build still uses Turbopack.
- **No shadcn init** — too much boilerplate for a small project. Hand-rolled primitives with consistent patterns.

## Single-user gate

Clerk middleware redirects unauthenticated traffic to `/sign-in`. After sign-in, `src/lib/owner-check.ts` compares the user's email against `APP_OWNER_EMAILS`. Non-owners hit `/forbidden` with a sign-out button. Bootstrap rule: empty `APP_OWNER_EMAILS` lets first signup through (set the env var immediately after).

## Telegram security

- Webhook secret in `x-telegram-bot-api-secret-token` header, checked first thing
- Chat ID allowlist via `TELEGRAM_ALLOWED_CHAT_IDS`
- Unauthorized chats get a polite "your chat ID is X" reply (helps onboarding, no info leak — chat IDs aren't secret)
- HTML mode replies escape user input (originally inconsistent, fixed during security audit)

## Same-origin guard

`/api/transcribe` and Server Actions check `Origin` against an allowlist (production domain + localhost + Tailscale + `*.vercel.app` previews). Defense in depth alongside Clerk. The real auth boundary is Clerk middleware.

## Deploy + migrations

Push to `main` → Vercel auto-deploys via the GitHub integration. The build script is `drizzle-kit migrate && next build` (since #33), so any pending Drizzle migrations are applied automatically before the new code goes live. The `todo.warmwetcircles.com` alias points at the latest production deployment.

**Schema-drift gotcha** — Drizzle's `db.select().from(table)` projects every column declared in `src/lib/db/schema.ts`. If schema in code drifts ahead of the prod DB, every page that hits that table 500s with `Failed query: select…`. This caused a hard outage on 2026-05-03 (#32) — prod was missing migrations 0003 + 0004 because nothing was applying them. Workflow now: edit `schema.ts` → `pnpm db:generate` → commit the new `drizzle/NNNN_*.sql` alongside the schema change → push.

## Useful commands

```bash
# Local dev (webpack, binds to 0.0.0.0)
pnpm dev

# Generate a migration after editing schema.ts (build will apply it)
pnpm db:generate

# Apply migrations manually (rare — build does this)
pnpm db:migrate

# Pull env from Vercel
vercel env pull .env.local

# Tail prod logs
vercel logs https://todo.warmwetcircles.com
```

## Open issues

For live status: `gh issue list --repo calvinmoltbot/mytodo`.

Today's session (2026-04-29) closed #1-9, #11, #13, #14, #15, #17, #19. Still open: #12 design depth, #16 Whisper guardrail, #18 recurring tasks (LLM detects them, no scheduler yet), #20 seasonal view, #21 weather, #22 PWA install, #23 daily digest, #24 Blob privacy.
