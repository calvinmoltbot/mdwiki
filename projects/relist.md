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

**Current state (2026-04-10):** Core features functional with real Neon DB. Inventory tracker (full CRUD with photos, edit dialog, status transitions), AI Description Generator (OpenRouter multi-model), and Profit Dashboard (charts, breakdowns by category/source/month) are built. 75 tests across 7 files. DATABASE_URL on Vercel production.

Issues tracked in `calvinmoltbot/relist`:

| # | Feature | Status |
|---|---------|--------|
| #1 | Connect Neon DB / Drizzle migrations | **Closed** |
| #2 | Test AI description e2e with real photo | Open |
| #3 | Inventory item edit dialog | **Closed** |
| #4 | Fix status dropdown (base-ui render prop) | **Closed** |
| #5 | Profit Dashboard | **Closed** |
| #6 | Deal Finder | Open |
| #7 | Price Intelligence DB | Open |
| #8 | Morning Dashboard | Open |

Priority from Lily's answers (original issue refs from vinted repo):
1. **AI Description Generator** (#18) — built, needs real photo testing (#2)
2. **Inventory Tracker** (#5) — fully functional with real DB, photos, edit dialog
3. **Profit Dashboard** (#6) — built with charts, category/source/month breakdowns
4. **Price Intelligence DB** (#16) — foundation for deal finder
5. **Deal Finder** (#15) — her dream feature
6. **Morning Dashboard** (#20) — daily briefing

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
- **shadcn/ui uses base-ui/react, NOT Radix** — `render` prop instead of `asChild`. SheetTrigger/DialogTrigger `render` prop is unreliable for custom components — use controlled open state with plain Button onClick instead (see pattern: `add-item-dialog.tsx`)
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
