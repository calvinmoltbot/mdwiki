---
title: letterboxd-tracker
tags: [project, vercel, neon, nextjs, plex, transmission]
created: 2026-04-25
updated: 2026-04-28
status: active
related:
  - ../systems/vercel-deploy-gotchas.md
  - ../systems/vercel-neon-marketplace.md
---

# letterboxd-tracker

Web app showing which films on Calvin's Letterboxd watchlist are available to stream in the UK. Cross-references with local Plex library and integrates with Transmission for downloads. Has a Discover tab that surfaces hot torrents with quality / streaming-availability awareness — runs only on the Mac Mini, never on Vercel.

- **Repo:** `calvinmoltbot/letterboxd-tracker` (also `calvinmoltbot/lb-tpb-scraper`, private — the Bun service that scrapes TPB for the Mini-only Discover build)
- **Production (Vercel):** https://letterboxd.warmwetcircles.com — Watchlist / Plex Library / Downloads
- **Local (Mini):** http://100.90.11.37:3030 — adds Discover tab, behind the Mini's launchd
- **Vercel project:** `prj_Ju85K4N6nkXtUXvWHPjSGkYX1doN` (calvin-orrs-projects, Pro plan, lhr1)
- **DB:** Neon project `dawn-brook-63693668` (single branch `neondb`, dev + prod share it)

## Stack

Next.js 15 (App Router) · Tailwind v4 · shadcn/ui · Drizzle ORM · NeonDB Postgres · cheerio (for Letterboxd HTML scraping) · TMDB API · Bun (for the lb-tpb-scraper service on the Mini).

## Architecture

**Two-track deployment**:

- `main` branch → Vercel-deployed app (clean, no tracker code)
- `feature/discovery-via-mini-proxy` branch → **Mini-only**, runs `next start -p 3030` under launchd. Adds the Discover tab and `/api/tpb` route. Never deployed to Vercel — `vercel.json` on both branches sets `git.deploymentEnabled["feature/discovery-via-mini-proxy"] = false`.

**Discover pipeline:**

```
TPB scrape (Bun, lb-tpb-scraper:7001 on Mini)
  → /api/tpb on Mini (token-jaccard match → TMDB enrich → Plex cross-ref → providers fetch)
  → in-memory cache keyed on scrapedAt (1h TTL)
  → DiscoverPage UI (chips, hover panel, Transmission controls)
```

**Watchlist pipeline (the Vercel app):**
Letterboxd HTML scrape → TMDB search (resolve IDs) → TMDB Watch Providers (UK) → cache in NeonDB → serve via API. Plus Plex library push for cross-referencing local copies, plus Transmission RPC proxy for torrent management.

API route count is ~9 unique routes — well under any plan limit because Next.js + Vercel auto-dedupe routes into shared function bundles via symlinks at build time.

## Transmission lives on the MacBook (2026-04-28)

Transmission was moved off the Mini onto Calvin's MacBook so its peer traffic
can route through Surfshark VPN without risking the Mini's Tailscale stack.

- **Daemon:** runs on `calvins-macbook-air-3.tail9422ba.ts.net:9091`, RPC auth
  off, RPC whitelist `100.*.*.*` (Tailscale CGNAT).
- **VPN binding:** Transmission's `bind-address-ipv4` is the Surfshark
  WireGuard interface (`utun5`, `10.14.0.2`). Public peer IP via VPN, real IP
  hidden.
- **Mini-side wiring:** `TRANSMISSION_URL` in `.env.local` (gitignored) on the
  discovery branch points at the MacBook hostname above. Default in
  `lib/transmission.ts` still references the Mini for back-compat — only the
  env var is live.
- **Post-download flow:** Transmission's `script-torrent-done` hook on the
  MacBook rsyncs completed files over SSH/Tailscale to
  `admin@mini-server.tail9422ba.ts.net:'/Volumes/SD Card/Plex/Movies/'` (the
  external 512 GB ExFAT SSD attached to the Mini, mislabelled "SD Card").
  Hook log at `~/Library/Logs/transmission-done.log` on the MacBook.
- **Trade-off:** downloads only progress while the MacBook is awake. Calvin
  accepted this — home-only use, fibre is fast, ratio not a concern.
- **Why not Surfshark on the Mini directly:** Surfshark's macOS client
  kill-switch will break Tailscale (it's the only way to reach a headless
  Mini). See `/Users/admin/shared/markviewer/letterboxd-tracker/2026-04-28-surfshark-vpn-research.md`
  for the full research, including the Gluetun-in-Docker fallback option that
  was rejected as too heavy.

## Mini operational state

Two launchd agents, both `KeepAlive=true`:
- `com.calvin.lb-tpb-scraper` — Bun service at `~/Dev/Projects/lb-tpb-scraper`, `:7001`, exposes `/health` + `/movies`
- `com.calvin.letterboxd-tracker-local` — `next start --hostname 0.0.0.0 -p 3030` from the discovery branch checkout

### Dev cycle (one-way main → discovery)

- **Vercel-side work:** branch off `main`, PR back to `main`. Normal flow.
- **Discover-side work:** branch off `feature/discovery-via-mini-proxy`, PR back to that branch. Never to main.
- **Pull main's improvements into the discovery branch:** run `./scripts/sync-from-main.sh` from the discovery branch — fetches, merges `origin/main`, pushes, runs `mini-rebuild.sh`. The script refuses to run from any other branch.
- **After local edits to discovery branch:** `./scripts/mini-rebuild.sh` (uses `CI=true pnpm install` for non-TTY contexts).

## User context

There's **no real auth** — identity is derived from the URL `/:username`. Multi-user via separate watchlist usernames in the `films` table:

- `lilyorr` — 421 films (heavy user, primary canonical)
- `calvinmolt` — 2 films
- `calvinorr` — 0 films (orphan; should redirect)

Shared infrastructure: Plex library (one server), Transmission (one client), TPB scrape (global). Per-user data: watchlists, Plex match cross-reference, Watchlist↔Discover badge.

`SUBSCRIBED_IDS` (Netflix / Apple TV+ / Prime / Disney+) and the 3 GB size ceiling are currently hardcoded as Calvin's preferences. Tracked as backlog: per-user prefs.

## Gotchas baked into the codebase

- **No `.npmrc`** — must stay that way. A `.npmrc` setting `store-dir=${HOME}/...` silently broke Vercel deploys for 34 days. See [vercel-deploy-gotchas](../systems/vercel-deploy-gotchas.md) for the full post-mortem.
- **Production env vars must be added with `--no-sensitive`** when scripted via Vercel CLI v51, otherwise they store as `""`.
- **No piracy-adjacent URLs in the Vercel bundle.** All TPB code (scraper + `/api/tpb` route + Discover UI) lives only on the Mini-only branch and is gated by the Vercel deploy-disable rule.
- **`pnpm db:push` can be misleading** when schema and DB are in transient mismatch (e.g. mid-migration with column added in code but not yet in DB). Run `npx tsx scripts/check-films-duplicates.ts` and `inspect-films-constraints.ts` to verify state before assuming drift is real. (Apr 26 session: the films_user_slug "drift" turned out to be a phantom — constraint already existed.)
- **Custom domain** `letterboxd.warmwetcircles.com` is attached to the project; aliases auto-update on each prod deploy.

## Backlog

- #34 — Redirect or land sensibly when `/:username` has no data
- #35 — Per-user preferences (subscriptions, size limit, hide list) — localStorage namespacing the cheap path
- #36 — Decide on real auth (Clerk / Auth0)

## Related sessions

- 2026-04-25 — silent Vercel deploy failures, 6-hour debug, `.npmrc` root cause, Pro upgrade, project delete+reimport, env var cleanup. Distilled into [vercel-deploy-gotchas](../systems/vercel-deploy-gotchas.md).
- 2026-04-26 — built out the entire Discover tab on the Mini-only branch (16 PRs across 14 issues): token-jaccard matcher, freshness UI, filters, Transmission live progress + controls, hide-forever, size/quality chips, hover card, Plex resolution badge w/ upgrade indicator, streaming-availability chips, Watchlist↔Discover badge. Used parallel subagent worktrees for #30/#31.
