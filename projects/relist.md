---
title: ReList — Vinted Reseller SaaS
created: 2026-04-09
updated: 2026-04-09
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

## Phase 2 (relist repo) — Planned

Priority order based on Lily's actual answers:

1. **AI Description Generator** (#18) — her #1 pain point
2. **Inventory Tracker** (#5) — must beat her Notion spreadsheet
3. **Profit Dashboard** (#6) — 65% margin target on £3k/mo
4. **Price Intelligence DB** (#16) — foundation for deal finder
5. **Price Estimator** (#11) — "how much should I sell this for?"
6. **Deal Finder** (#15) — her dream feature: live feed of flippable listings
7. **Morning Dashboard** (#20) — she'd use it as a morning routine

Issues are tracked in `calvinmoltbot/vinted` repo (Phase 2 label).

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
- **Vinted has no public API** — mobile API at `/api/v2/` has no DataDome protection (key insight from Apify research)
- **Deal finder needs proxies** — residential, geo-matched to UK, ~£20/month
- **TLS fingerprinting** — vanilla HTTP clients get detected, need `curl_cffi` or `tls-client`
- **Sold listings not accessible** — infer sales by tracking items that disappear

## Research

Detailed research saved in markviewer:
- `vinted/2026-04-09-vinted-data-research.md` — API access, anti-bot, Chrome extension approach
- `vinted/2026-04-09-relist-prd.md` — PRD v2 based on Lily's actual answers
- `vinted/2026-04-09-lily-answers-summary.md` — comparison of Calvin's assumptions vs Lily's reality
