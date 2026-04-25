---
title: herbarium-hq — Business HQ for Herbarium Dyeworks
created: 2026-04-25
updated: 2026-04-25-pm
status: active
tags: [herbarium, debbie, business-hq, nextjs]
related:
  - ../systems/vercel-neon-marketplace.md
---

# herbarium-hq

Business HQ for **Debbie's natural-dye yarn business** (Herbarium Dyeworks). Companion to `herbarium-finance` — different DB, different purpose: **strategic view** (asset value, plan, stock, channels) rather than transaction bookkeeping.

For live feature state, use `gh issue list --repo calvinmoltbot/herbarium-hq` — not this page.

## URL & infra

- Live: **https://hq.herbariumdyeworks.uk**
- Sibling: `finance.herbariumdyeworks.uk` (the bookkeeping app)
- Repo: `calvinmoltbot/herbarium-hq`
- Vercel project: `calvin-orrs-projects/herbarium-hq`
- DB: Neon Postgres, provisioned via [Vercel Marketplace](../systems/vercel-neon-marketplace.md)

## Tech

- Next.js 16 App Router (webpack — Turbopack disabled for Tailscale dev)
- React 19, TypeScript, Tailwind v4 (no config; `@theme inline` in globals.css)
- Drizzle ORM
- `@supabase/supabase-js` for cross-app read from `herbarium-finance`'s Supabase

Dev server: `pnpm dev` runs `next dev --hostname 0.0.0.0 --webpack` so MacBook Chrome can hit `http://100.90.11.37:3000` over Tailscale.

## Architecture

**One-way cross-app read** — herbarium-hq reads `herbarium-finance`'s Supabase via service role key (server-side only, `src/lib/finance/adapter.ts`), never writes back. Cash position + recent revenue come from finance; everything else (stock, equipment, sales, plan) is hq's own Neon.

**Asset register pattern** — three CRUD pages share an identical structure:

```
src/app/<entity>/
  page.tsx              list + rollup tile + inline delete
  new/page.tsx          add form
  [id]/edit/page.tsx    edit form
  _form.tsx             shared form component
  actions.ts            server actions: create/update/delete
src/lib/db/<entity>.ts  data-access (queries + value rollup helper)
```

Used for `equipment`, `stock`, `sales`. Materials (issue #21) follows same shape.

**Dashboard tile rollup** — each entity exposes a `getXxxTotalValue()` from its data-access module; `/` calls them all in parallel and sums for "Total business value". Fallbacks to 0 / `—` if the table's empty.

## Finance schema gotcha

`herbarium-finance`'s `transactions` table uses `transaction_date` (not `date`) and signs amounts via a `type` column (`expenditure | income | capital`) — `amount` is always unsigned. The scaffold's adapter assumed signed amounts and a `date` column; fixed in PR #16. If extending the adapter, remember:

```sql
-- Cash = income + capital − expenditure
-- Revenue (last N days) = where type='income' AND transaction_date >= ?
```

## Auth

Currently **single-tenant, no auth**. Plan (#14) is to reuse Supabase Auth from `herbarium-finance` so Debbie has one login across both apps. Service-role finance reads stay server-side regardless.

## Supabase keys

Using **legacy** `service_role` JWT for the cross-app read. New Publishable/Secret system migration filed as #22 — not urgent, legacy keys still work.

## Open questions tracked as issues

- Plan editor UX: Vinted-style one-question-per-screen vs open textarea (#6)
- Revolut → in_person_sales reconciliation (#11)
- Stock depreciation model (#13 margin tool covers part of it)
- Shopify integration: direct Admin API vs Shopify AI toolkit (#10)
- Mobile: responsive vs separate phone-first capture (#15)

## Status snapshot (2026-04-25 — PM)

MVP shipped. Now in dashboard-improvement mode — surfacing finance data for strategic interpretation (not redoing the bookkeeper view). PR #25 added anchored period filters, hero KPI strip with sparklines + run-rate, sign-flip-aware deltas. Plan + research in `markviewer/herbarium-hq/2026-04-25-dashboard-improvement-plan.md`. Channel mix + plain-English summary tracked as #26 (PR-B).

## Design principles (from this session)

The framing that drives every dashboard decision:

- **Interpretation, not data** — the bookkeeping app has the numbers; hq exists to make them speak
- **Three lenses** — held value (assets), operating (am I making money?), invested to date (what did it take to get here?). A £280 dye pot is not a £280 loss
- **Anchored periods** — every period filter resolves against the latest finance transaction date, not "today". Stale imports never produce empty-looking dashboards
- **Plain language over jargon** — "20p kept of every £1" beats "operating margin %"
- **What to skip**: EBITDA, gross-vs-net tables, cohort/LTV analytics, gauge dials, multi-month cash forecasting — all alienating or noise-dominated at £8K/yr scale

## Anchor pattern (reusable)

For any read-only dashboard against a data source that may be stale: query for the latest record's date and use *that* as the "now" anchor for period filters. Surface the anchor visibly to the user. Pattern lives in `src/lib/finance/periods.ts`.
