---
title: ReList — Vinted Reseller SaaS
created: 2026-04-09
updated: 2026-04-10
tags: [vinted, saas, reselling, lily]
---

# ReList

SaaS tool for Vinted resellers, built for Calvin's daughter Lily (22) who runs **lily.dor** selling vintage women's clothing, books, shoes, and handbags.

## Architecture

**Two separate repos/deploys:**

| Repo | Purpose | URL | Status |
|------|---------|-----|--------|
| `calvinmoltbot/vinted` | Phase 1 — Business plan capture (Typeform-style) | vinted.warmwetcircles.com | Complete |
| `calvinmoltbot/relist` | Phase 2 — SaaS app (inventory, deals, pricing) | relist.warmwetcircles.com | Not started |

## Phase 1 (vinted repo) — Complete

Interactive Typeform-style flow capturing Lily's business plan across 5 sections:
- Core Business, Goals & Financials, Operations, Growth & Brand, Your Dream Tool
- Config-driven steps (DB-backed, editable via /admin)
- Feedback system per question
- Auto-save to Neon Postgres + localStorage
- Dashboard summary, edit pages, financial projections, what-if explorer

**Tech:** Next.js 16, Tailwind, shadcn/ui, Framer Motion, zustand, Neon Postgres, Vercel

## Phase 2 (relist repo) — In Progress

**Tech:** Next.js 16.2.3 (App Router, webpack mode), React 19, Tailwind v4, shadcn/ui (base-ui/react), Framer Motion, Zustand 5, Drizzle ORM + Neon Postgres, OpenRouter for AI vision models.

**Current state (2026-04-10):** All core features built. Morning Dashboard, Inventory (full CRUD, photos, AI descriptions, xlsx/JSON import), Describe page (polished, lightbox, dropdowns), Profit Dashboard (revenue targets, charts). Price data API + Chrome extension scaffold built (zero-cost approach — passive data collection from Lily's browsing). 89 tests across 9 files. Lily's 50 real sold items imported.

Issues tracked in `calvinmoltbot/relist`:

| # | Feature | Status |
|---|---------|--------|
| #1 | Connect Neon DB / Drizzle migrations | **Closed** |
| #2 | Test AI description e2e with real photo | **Done** (working) |
| #3 | Inventory item edit dialog | **Closed** |
| #4 | Fix status dropdown (base-ui render prop) | **Closed** |
| #5 | Profit Dashboard | **Closed** |
| #6 | Deal Finder | Open — Chrome extension approach (zero cost) |
| #7 | Price Intelligence DB | Open — price data API built, needs extension testing |
| #8 | Morning Dashboard | **Closed** |
| #9 | Price Estimator | Open — local price data + free LLM |
| #10 | Revenue target settings (configurable) | Open |
| #11 | Cross-platform listing tracker | Open — low priority |
| #12 | Clean up mock store | **Closed** |
| #13 | Chrome extension scaffold | **Closed** — built, needs DOM selector testing |
| #14 | Price data API endpoints | **Closed** |
| #15 | Help system and documentation | Open |

**Research saved:** `markviewer/relist/2026-04-10-deal-finder-price-estimator-approaches.md`

**Strategy shift:** Dropped Apify (too expensive for a starting business). Using zero-cost Chrome extension that passively collects price data from Lily's normal Vinted browsing.

Priority:
1. **Help system / docs** (#15) — next session
2. **Deal Finder** (#6) — test Chrome extension on real Vinted pages, tune DOM selectors
3. **Price Estimator** (#9) — needs price data from extension first
4. **Revenue target settings** (#10) — currently hardcoded to £3k/mo

## Key Business Context

- **Shop:** lily.dor on Vinted
- **Revenue target:** £3,000/month (6-month goal: £1,500)
- **Margin target:** 65%
- **Hours:** 25/week
- **Startup investment:** £220, monthly costs £50
- **Current tools:** Vinted's own tools + Notion spreadsheet
- **Primary device:** Both phone and laptop
- **Brand importance:** 2/5 (selling > brand building)

## Gotchas

- **Never use Supabase** — Calvin's rule. Use Neon Postgres (free via Vercel integration)
- **shadcn/ui uses base-ui/react, NOT Radix** — `render` prop instead of `asChild`. All trigger/close `render` props are unreliable — use controlled open state instead. Dialogs switched from Sheet (narrow) to Dialog (wide centered modal) for add/edit forms.
- **Image resize on upload** — Phone photos are 5-10MB as base64. Must resize client-side before storing/sending. `resizeImage()` in describe-store.ts scales to 1200px max, JPEG 70%. Resize at upload time, not send time.
- **Vinted scraper** — No API, no export. Claude CoWork built a browser console scraper (see `Data/vinted_scraper_handover.md`). ReList imports the output via xlsx or JSON paste. Scraper stays separate from the app.
- **Route file exports** — Next.js App Router route files can only export HTTP methods. Exporting constants (like AVAILABLE_MODELS) causes TS errors.
- **Next.js 16 `allowedDevOrigins`** — Required in `next.config.ts` for cross-origin dev access. Without it, client JS hydration silently fails when accessing from non-localhost (e.g. Tailscale IP). Symptom: page renders server HTML but no interactivity.
- **Next.js 16 default Turbopack** — Must pass `--webpack` to `next dev`. Turbopack has a bug where non-localhost connections hang (TCP connects but no HTTP response).
- **globalThis for HMR persistence** — In-memory mock data stores need `globalThis.__storeName` pattern to survive webpack HMR reloads. Module-level variables get reset.
- **OpenRouter for AI** — Using OpenAI SDK with `baseURL: https://openrouter.ai/api/v1`. 80x cheaper than direct Claude API. Key in `.env.local` as `OPENROUTER_API_KEY`.
- **Vinted has no public API** — mobile API at `/api/v2/` has no DataDome protection (key insight from Apify research)
- **Deal finder needs proxies** — residential, geo-matched to UK, ~£20/month
- **TLS fingerprinting** — vanilla HTTP clients get detected, need `curl_cffi` or `tls-client`
- **Sold listings not accessible** — infer sales by tracking items that disappear

## Research

Detailed research saved in markviewer:
- `vinted/2026-04-09-vinted-data-research.md` — API access, anti-bot, Chrome extension approach
- `vinted/2026-04-09-relist-prd.md` — PRD v2 based on Lily's actual answers
- `vinted/2026-04-09-lily-answers-summary.md` — comparison of Calvin's assumptions vs Lily's reality
