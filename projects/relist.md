---
title: ReList — Vinted Reseller SaaS
created: 2026-04-09
updated: 2026-04-23
tags: [vinted, saas, reselling, lily]
---

# ReList

SaaS tool for Vinted resellers, built for Calvin's daughter Lily (22) who runs **lily.dor** selling vintage women's clothing, books, shoes, and handbags.

## Architecture

Two separate repos/deploys:

| Repo | Purpose | URL |
|------|---------|-----|
| `calvinmoltbot/vinted` | Phase 1 — Typeform-style business plan capture (complete) | vinted.warmwetcircles.com |
| `calvinmoltbot/relist` | Phase 2 — SaaS app (inventory, deals, pricing) | relist.warmwetcircles.com |

For live feature state and priorities, use `gh issue list` — not this page.

Dashboard redesign (2026-04-12): "Today's Flow" kanban (Ship / Update / Review) on left, Performance panel on right with real 7-day sparkline + rolling 7-day-vs-weekly-target pace card. Daily-plan data folded into `/api/dashboard` for single round trip. Markup scratchpad (2026-04-13) sits under the kanban — cost input + markup slider with 100/150/200/300% presets → list price & profit.

Backup (2026-04-13): `GET /api/backup` dumps all tables as one JSON file (Settings → "Download backup"). Each download upserts `last_backup_at` in `user_settings`; Dashboard shows an amber nudge banner when it's been 14+ days or never (3-day localStorage dismiss).

Restore (2026-04-18, #28 / PR #29): Settings → "Restore From Backup" uploads a backup JSON, shows a row-count diff vs current DB, requires typed `RESTORE` confirmation, and auto-downloads a pre-restore snapshot to Downloads just before POSTing. `POST /api/backup/restore` wipes tables in FK-safe order (children first: transactions, expenses, watchItems, dealAlerts, priceData, priceStats, userSettings, then items) and re-inserts parent-first. **Gated off in `VERCEL_ENV=production`** until auth (#16) lands — read-only `GET /api/backup/restore` (returns current counts) is always allowed. **neon-http driver has no transactions**, so the wipe + reinsert runs sequentially — the client-side pre-restore download is the safety net if anything fails mid-way.

## Tech

Next.js 16.2.3 (App Router, **webpack mode — not Turbopack**), React 19, Tailwind v4, shadcn/ui (base-ui/react), Framer Motion, Zustand 5, Drizzle ORM + Neon Postgres, OpenRouter for AI vision.

## Business context

- Shop: **lily.dor** on Vinted
- Revenue target: £3k/month (6-month: £1.5k), 65% margin, 25h/week
- Startup £220, monthly costs £50
- Lily uses Vinted iPhone app primarily; ReList on laptop for tracking/analytics, phone for quick financial checks
- Brand importance low (2/5) — selling > brand building

## Gotchas

- **Never use Supabase** — Calvin's rule. Use Neon (free via Vercel integration).
- **shadcn/ui here uses base-ui/react, NOT Radix** — `render` prop instead of `asChild`. Trigger/close `render` props are unreliable; use controlled open state. Add/edit forms use Dialog (wide centered), not Sheet.
- **Next.js 16 defaults to Turbopack** — must pass `--webpack` to `next dev`. Turbopack hangs on non-localhost connections (TCP connects, no HTTP response).
- **`allowedDevOrigins` in next.config.ts** — required for Tailscale-IP dev access. Without it client JS hydration silently fails (server HTML renders, no interactivity).
- **Image resize on upload** — phone photos are 5–10MB base64. `resizeImage()` in describe-store.ts scales to 1200px / JPEG 70% *at upload time*, not send time.
- **Route files can only export HTTP methods** — App Router TS errors if you export constants from `route.ts`.
- **HMR persistence** — in-memory mock stores need `globalThis.__storeName` to survive webpack HMR.
- **OpenRouter for AI** — OpenAI SDK with `baseURL: https://openrouter.ai/api/v1`. ~80× cheaper than direct Claude. Key: `OPENROUTER_API_KEY`.
- **Vinted has no public API / export** — mobile `/api/v2/` has no DataDome but needs residential UK proxies (~£20/mo) and TLS-fingerprint-aware clients (`curl_cffi` / `tls-client`). Strategy shift (2026-04-10): dropped Apify, use zero-cost Chrome extension that passively collects price data from Lily's normal browsing. Sold listings aren't accessible — infer sales from items that disappear.
- **Sticky table headers** — `position: sticky` on a `<th>` resolves against the nearest scrolling ancestor. If the table wrapper has `overflow-x-auto`, sticky latches onto it (non-scrolling vertically) and the header never pins. Fix: the table wrapper itself must be the vertical scroll container (`flex-1 min-h-0 overflow-y-auto`), with the page using `h-full flex-col` so the wrapper can claim remaining height.
- **Vinted scraper lives separately** — browser console scraper built by Claude CoWork (`Data/vinted_scraper_handover.md`). Import pathway removed 2026-04-13 (Excel/CSV import + xlsx dep + Vinted-Scraper help group all deleted) — Chrome extension is now the only ingest path.
- **Item delete needs explicit FK cleanup** — `transactions.itemId` is `NOT NULL` with no `ON DELETE CASCADE`, and `expenses.itemId` / `watchItems.convertedItemId` are nullable refs. Any DELETE on `items` must first `DELETE FROM transactions WHERE item_id = ?` and null out the other two, otherwise Postgres rejects and the row appears to vanish client-side (optimistic update) but reappears on refetch. Bulk + single DELETE handlers both need this — see `src/app/api/inventory/[id]/route.ts` and `.../bulk/route.ts`. Always check `res.ok` on the client delete so failures don't disappear silently.
- **Base UI Menu error #31 = GroupLabel outside Group** — if `DropdownMenuLabel` (which wraps `Menu.GroupLabel`) isn't inside `DropdownMenuGroup` (`Menu.Group`), clicking the trigger throws `Base UI error #31` at runtime (message-minified in prod: "MenuGroupRootContext is missing"). Other minified error codes can be traced via `grep -rn "formatErrorMessage(N" node_modules/@base-ui/`.
- **Neon transfer cap = silent empty responses** — when a Neon project hits its monthly data-transfer allowance, queries don't error, they return empty. Inventory looked "wiped" and the dashboard zeroed out on 2026-04-17; data was intact, just unreadable until upgrade. Check the Neon console's billing page before assuming data loss. Upgrade path is Vercel-managed Launch tier (no separate bill — rolls into Vercel invoice).
- **Photos are base64 in Postgres** — `items.photo_urls` is `text[]` of `data:image/jpeg;base64,...` strings at 1200×1200 q70 (~300 KB each). Also a `thumbnail_url` column (200×200 q60, ~9 KB) added 2026-04-17; `/api/inventory` list returns **only** `thumbnailUrl` + `photoCount`, never the full array. Full photos come from `/api/inventory/[id]` for the edit dialog. New writes generate both via `src/lib/photos.ts`. Backfill script: `scripts/backfill-thumbnails.mjs` (idempotent via `WHERE thumbnail_url IS NULL`).
- **Thumbnail data URIs are served as binary by a dedicated route** (2026-04-23, PR #44). `/api/inventory` previously inlined the full base64 data URI per row → ~976 KB list payload. Now the query returns `'/api/inventory/thumb/' || items.id` and a new `GET /api/inventory/thumb/[id]` route decodes the stored base64 and serves `image/jpeg` with `Cache-Control: public, max-age=604800, immutable` + sha1 ETag. Browser caches each thumb independently; list payload drops to ~56 KB (−94%). Edit dialog fetches the full item on open to re-hydrate `vintedUrl` / `description` / `photoUrls` which are deliberately absent from the list query for the same bandwidth reason.
- **Extension is the only item-creation path** (2026-04-23, PR #45). Quick Log's "Add new item" was silently creating records without `vintedUrl` / photos / description — Lily's workflow is listing-first via the Chrome extension, she never manually creates items. Quick Log is now update-only (Sold / Shipped); searching for a non-existent item shows a callout pointing at the extension. Also deleted `POST /api/inventory/[id]/fetch-vinted` (server-side scraper — ToS risk) and the `FetchFromVinted` button on the edit dialog. Extension runs in Lily's logged-in Vinted session so DOM reads are user-initiated, not scraping.
- **Vinted URL lookups need normalisation** (2026-04-23, PR #46). The extension sends `window.location.href` to `/api/inventory/check`, but Vinted decorates URLs with tracking params (e.g. `?referrer=personal_profile` when Lily clicks through her own profile). Stored URLs are clean, so exact-match missed and the extension fell back to "Add / Watch for Flip" on her own items. Fix in `check/route.ts`: strip query + hash + trailing slash via `new URL()`, then match with `OR(eq(clean), like(clean||'%'))` as a safety net for legacy data.
- **Completeness scorer fields = 6 (2026-04-23)** — `brand(20) + category(15) + size(10) + description(20) + photos(20) + title(10) + vintedUrl(5) = 100`. `condition` deliberately omitted (Lily UX avoids it). Aging buckets tuned to Lily's 2-week ceiling: `0-3 / 4-7 / 8-14 / 15-21 / 22+` days (stockAtRisk sums 15+).
- **Dashboard + Inventory are async Server Components** (2026-04-18, PR #31). Query logic extracted to `src/lib/dashboard.ts` and `src/lib/inventory-query.ts`; page files are ~12-line shells that call those and pass data into `dashboard-client.tsx` / `inventory-client.tsx`. Zustand store got a `hydrate(items)` action and a `hydrated` flag so the first mount doesn't re-fetch from `/api/inventory`. `export const revalidate = 60` on both pages gives ISR. The `/api/dashboard` and `/api/inventory` routes still exist as thin wrappers for filter-change and mutation refreshes.
- **Cache-Control idiom for read endpoints** — `private, max-age=120, stale-while-revalidate=300` on Dashboard/Inventory, `private, max-age=300, stale-while-revalidate=600` on Profit, `private, max-age=60, stale-while-revalidate=180` on watch-items/expenses. Deliberately skipped on `/api/settings` — users expect saves to reflect immediately.
- **No Drizzle joins were used in the codebase until PR #31** — pattern was established via `leftJoin(priceStats, and(eq(...), eq(...)))` in `src/app/api/watch-items/route.ts` to replace an N+1. Result rows come back as `{ watchItem, stat }` tuples; dedup by `watchItems.id` using a Map with first-hit-wins after `orderBy(..., desc(priceStats.lastUpdatedAt))`.
- **Integration tests share the real Neon DB** — `tests/*.test.ts` use `import { db } from "@/lib/db"` directly. Vitest `test.env` doesn't propagate to workers in v4.1.x; the working pattern is `node --env-file-if-exists=.env.local ./node_modules/vitest/vitest.mjs` in the npm scripts (process env inherits through fork). Seeded rows must use distinctive names (e.g. "FK Test Item") and be swept by a bulk cleanup in `beforeAll`/`afterAll`/`beforeEach` — historical pattern of `.reverse()`ing `cleanupFns` silently left rows on the live DB when a delete was expected to fail (FK constraint tests) and the catch swallowed it. Long-term fix is a separate test Neon project, not done yet.

## Research

- `markviewer/vinted/2026-04-09-vinted-data-research.md` — API access, anti-bot, extension approach
- `markviewer/vinted/2026-04-09-relist-prd.md` — PRD v2 based on Lily's actual answers
- `markviewer/vinted/2026-04-09-lily-answers-summary.md` — Calvin's assumptions vs Lily's reality
- `markviewer/relist/2026-04-10-deal-finder-price-estimator-approaches.md`
- `markviewer/relist/2026-04-11-financials-upgrade-plan.md`
- `markviewer/relist/2026-04-18-nav-lag-audit.md` — nav-lag root cause + tiered plan (Phases 1–3 shipped in PR #31)
