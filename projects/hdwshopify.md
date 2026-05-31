---
title: hdwshopify — Herbarium Dyeworks store analytics dashboard
created: 2026-05-31
updated: 2026-05-31
status: active
tags: [shopify, ga4, nextjs, vercel, analytics, google-auth, mailchimp]
related:
  - ../systems/ga4-data-api-on-vercel.md
  - ../systems/gcloud-headless-auth.md
  - ../patterns/nextjs-single-user-google-auth.md
  - ../systems/vercel-deploy-gotchas.md
---

# hdwshopify

A private **analytics dashboard for the Herbarium Dyeworks Shopify store** — a
plain-English read on how the shop is doing that the store's Admin doesn't surface
well. Shopify's Admin API gives commerce data (revenue, orders, products,
customers) but no visitor/traffic analytics, so GA4 fills that gap.

Local path: `~/Dev/Projects/hdwshopify`. Repo: `calvinmoltbot/hdwshopify`.
Live: **dashboard.herbariumdyeworks.uk** (Vercel project `hdwshopify`, team
calvin-orrs-projects). For live feature state use
`gh issue list --repo calvinmoltbot/hdwshopify` — not this page.

## Stack

**Next.js 16 + React 19** (App Router), server-only data libs, deployed on Vercel.
Note Next 16 quirks: Turbopack is the default bundler but breaks non-localhost dev
over Tailscale — run dev with `--hostname 0.0.0.0 --webpack` and set
`allowedDevOrigins` to the Mini's Tailscale IP. See [[ga4-data-api-on-vercel]] for
the GA4 client wiring.

## Pages (`src/app/(app)/`)

Overview (at-a-glance health) · Sales (revenue, products, regions) · Traffic (who
visits & from where) · Journey (visit→purchase GA4 event funnel) · Customers
(retention & loyalty) · Products (status & stock) · Newsletter (Shopify ↔ Mailchimp
list curation).

## Two data sources

- **Shopify Admin API** (`src/lib/shopify.ts`) — server-only client, reads
  `SHOPIFY_SHOP` (`hdw.myshopify.com`), `SHOPIFY_ADMIN_TOKEN` (`shpat_…`),
  `SHOPIFY_API_VERSION`. Drives commerce metrics. ⚠️ The store's plan **denies API
  access to customer email/PII** — customer reconciliation works off a manual admin
  CSV export, not the API.
- **GA4 Data API** (`src/lib/ga4.ts`, `@google-analytics/data`) — traffic + the
  journey funnel. Events flow via a Shopify **custom web pixel** ("hdwclaude"); data
  only began accruing ~2026-05-20. Auth is a property-owner **user refresh token**,
  not a service account (GA4 rejects SA emails). Full pattern + the recurring
  token-expiry gotcha: [[ga4-data-api-on-vercel]].

## Auth

**Single-user Google OAuth** gate (`src/lib/auth/`) — only allow-listed emails get
in, via the `ALLOWED_EMAIL` env var (comma-separated list). Pattern doc:
[[nextjs-single-user-google-auth]].

## Known gotchas / quirks

- **GA4 token dies every 7 days** unless the OAuth consent screen is published to
  production — now fixed (in production as of 2026-05-31). Two credential stores
  (local ADC file + Vercel env) drift independently and both need updating on a
  re-mint. Full detail: [[ga4-data-api-on-vercel]].
- **Many recent orders are genuine £0 orders** — free-ticket reservations, not a
  bug. Revenue queries use `currentTotalPriceSet`.
- **The web pixel is sandboxed** (no DOM access), so it can't track scroll/click
  depth — that needs a theme-level script, not the pixel (open backlog item).
- **Customer PII is blocked** at the Shopify plan level (see Shopify source above).

## Newsletter reconciliation

`/newsletter` curates the Shopify ↔ Mailchimp mailing list. It expects **one
Mailchimp export with a Status column**; Mailchimp's UI splits exports by status
(no Status column), so those must be merged before import.
