---
title: letterboxd-tracker
tags: [project, vercel, neon, nextjs]
created: 2026-04-25
updated: 2026-04-25
status: active
related:
  - ../systems/vercel-deploy-gotchas.md
  - ../systems/vercel-neon-marketplace.md
---

# letterboxd-tracker

Web app showing which films on Calvin's Letterboxd watchlist are available to stream in the UK. Cross-references with local Plex library and integrates with Transmission for downloads.

- **Repo:** `calvinmoltbot/letterboxd-tracker`
- **Production:** https://letterboxd.warmwetcircles.com
- **Vercel project:** `prj_Ju85K4N6nkXtUXvWHPjSGkYX1doN` (calvin-orrs-projects, Pro plan, lhr1)
- **DB:** Neon project `dawn-brook-63693668` (single branch `neondb`, dev + prod share it)

## Stack

Next.js 15 (App Router) · Tailwind v4 · shadcn/ui · Drizzle ORM · NeonDB Postgres · cheerio (for Letterboxd HTML scraping) · TMDB API.

## Architecture

Letterboxd HTML scrape → TMDB search (resolve IDs) → TMDB Watch Providers (UK) → cache in NeonDB → serve via API. Plus Plex library push for cross-referencing local copies, plus Transmission RPC proxy for torrent management.

API route count is ~9 unique routes — well under any plan limit because Next.js + Vercel auto-dedupe routes into shared function bundles via symlinks at build time.

## Gotchas baked into the codebase

- **No `.npmrc`** — must stay that way. A `.npmrc` setting `store-dir=${HOME}/...` silently broke Vercel deploys for 34 days. See [vercel-deploy-gotchas](../systems/vercel-deploy-gotchas.md) for the full post-mortem.
- **Production env vars must be added with `--no-sensitive`** when scripted via Vercel CLI v51, otherwise they store as `""`. Same wiki page covers it.
- **No piracy-adjacent URLs in the Vercel bundle.** TPB scraping (Discover tab) was removed for this reason; see [issue #1](https://github.com/calvinmoltbot/issues/1) for the plan to reintroduce it as a Mac Mini-hosted service that the Vercel `/api/tpb` route proxies via Tailscale.
- **Custom domain** `letterboxd.warmwetcircles.com` is attached to the project; aliases auto-update on each prod deploy.

## Open work

- Issue #1: Bring back Discover tab via Mac Mini scraper proxy

## Related sessions

- 2026-04-25 — silent deploy failures, 6-hour debug, `.npmrc` root cause, Pro upgrade, project delete+reimport, env var cleanup. Distilled into [vercel-deploy-gotchas](../systems/vercel-deploy-gotchas.md).
