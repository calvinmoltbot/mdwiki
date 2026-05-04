---
title: relist-saas — multi-tenant rebuild for friends and family
created: 2026-05-02
updated: 2026-05-04
tags: [vinted, multi-tenant, friends-and-family, nextjs, neon, clerk]
---

# relist-saas

Multi-tenant version of ReList, built for Calvin's friends and family who sell on Vinted. **Separate product** from the legacy `relist` app — that one is Lily's live livelihood and we never touch it.

## The two-app rule (do not violate)

| App | Path / repo | Status | What we do here |
|---|---|---|---|
| Lily's live ReList | `~/Dev/Projects/relist` · `calvinmoltbot/relist` | Production, working well | Read-only. If we spot a bug, raise an issue in that repo. **No edits, no commits, no pushes.** |
| relist-saas (this) | `~/Dev/Projects/relist-saas` · `calvinmoltbot/relist-saas` | Active build | Free for friends-and-family, multi-tenant, no plans to charge. |

PRDs and planning docs live at `~/shared/markviewer/relist-saas/` (current PRD: `2026-05-02-prd.md`). The old `~/shared/markviewer/relist/` and `~/shared/markviewer/vinted/` folders are historical artefacts — don't update them, supersede with new docs in `relist-saas/`.

## Architecture invariants

- **Next.js 16 + App Router + webpack mode** (not Turbopack — has localhost-bind issues over Tailscale; always pass `--webpack` to `next dev`).
- **Auth**: Clerk for web sessions, Bearer tokens against `api_keys` for the extension. `getUserId(req)` resolves either path; routes catch `UnauthorizedError` and return 401.
- **Sign-up gated by Clerk allowlist.** Public registration is closed — Calvin adds an email in the Clerk dashboard to grant access.
- **Tenancy invariant**: every domain table carries `user_id`. Indexes lead with `user_id`. All user-facing reads go through `userScope(userId)` in `src/lib/db/scoped.ts` — never the bare `db` import. `scripts/tenancy-check.mjs` enforces this.
- **Deploy**: push `main` → Vercel auto-deploys. Postbuild migrate runs Drizzle migrations.

## Constraints

- **Hosting cap: $5/month total** across Neon + Vercel + Clerk. Drives "BYO key for LLMs" and "users feed data via the extension, we don't run scrapers" defaults.
- **Free for users, forever.** No billing UI, no paid tiers. If a feature requires shared paid infra, default to BYO or park it.
- **Vinted-only.** Same scope as legacy — no eBay/Depop.

## Status (2026-05-04)

Design system foundation in place + Dashboard rebuilt + Lily's 120-item backup imported as sample data + DB perf trap fixed. **8 page-rebuild PRs queued** as issues #23–#31. Production lives at **`relist-saas.warmwetcircles.com`**.

Phase 2 deferred backlog complete. Live surfaces:

| Path | What it does |
|---|---|
| `/dashboard` | Headline tiles, top profit table |
| `/inventory` (+ detail) | Per-user CRUD, status transitions, photo upload + grid |
| `/profit` | Date-preset filtered profit aggregation |
| `/health` | Cadence, completeness, aging, dead stock, refresh queue |
| `/bestsellers` | Time-to-sell breakdown across attributes |
| `/plan` | Today's tasks (ship / update / reprice / photo) |
| `/expenses`, `/market` | Per-user ledger + price-data view |
| `/settings/api-keys` | Generate / revoke / delete personal Bearer tokens |
| `/settings/backup` | Per-user JSON export + REPLACE-confirmed restore |
| `/watch`, `/describe` | Coming Soon stubs (parked) |

**Coming Soon stubs in nav**: `/watch`, `/describe`. Both reserved slots only — no functionality yet.

**Parked indefinitely**: Price Estimator (inline on item edit), Deal Finder, Billing.

For live feature state, use `gh issue list --repo calvinmoltbot/relist-saas` — not this page.

## Where to look

- **PRD**: `~/shared/markviewer/relist-saas/2026-05-02-prd.md` — supersedes everything in `~/shared/markviewer/relist/` and `~/shared/markviewer/vinted/`.
- **Memory rules** (load automatically per session): relist isolation, $5 hosting cap, Clerk allowlist gate.
- **Schema**: `src/db/schema.ts`. Migrations in `drizzle/`.
- **DB access surface**: `src/lib/db/scoped.ts` (`userScope`).
- **Auth**: `src/lib/auth/getUserId.ts`.

## Gotchas

- The `next dev` script must include `--webpack`; Turbopack hangs on non-localhost connections (TCP establishes but no HTTP response).
- Drizzle config loads `.env.local` first (Vercel CLI writes there) — don't rely on `.env`.
- `next.config.ts` allows the Mac Mini's Tailscale IP as a dev origin so HMR + Clerk client iframes work over SSH.
- `globals.css` forces light scheme until a real theming pass — the boilerplate `prefers-color-scheme` rule was removed because it produced white-on-cream chaos.
- Photos travel as base64 data URIs in `items.photoUrls` / `items.thumbnailUrl`. **Never `db.select().from(items)` without specifying columns** — the table is ~46 MB after Lily's import and pulling the blobs slowed Health to 20 seconds. Use explicit slim columns or derive `hasThumbnail` boolean / `photoCount` integer in SQL. Item-detail and the photo/backup APIs are the only places that actually need the bytes. (Issue #31 tracks the Vercel Blob migration that retires this trap permanently.)
- **Clerk dev instances assign different userIds per Vercel domain.** Signing in to `relist-saas.warmwetcircles.com` and to a `relist-saas-git-...vercel.app` preview produced two different `user_xxx` IDs for the same email. Always test against the canonical subdomain only — never share preview URLs with Calvin. The Lily-data importer (`scripts/import-from-lily.mjs --user-id <clerk_id>`) must be run with the user's canonical subdomain userId.
- **Domain map** (also in `AGENTS.md` for in-session reference): `relist-saas.warmwetcircles.com` is THIS app's only canonical URL; `relist.warmwetcircles.com` is **Lily's live production site** — read-only forever; `vinted.warmwetcircles.com` is Lily brainstorming, also off-limits.
