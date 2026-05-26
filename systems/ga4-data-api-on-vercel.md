---
title: GA4 Data API on Vercel — refresh-token env creds
tags: [ga4, google-analytics, vercel, auth, adc, oauth, gotcha]
created: 2026-05-26
updated: 2026-05-26
status: active
sources:
  - hdwshopify/src/lib/ga4.ts
related:
  - ./gcloud-headless-auth.md
  - ./vercel-deploy-gotchas.md
  - ../patterns/nextjs-single-user-google-auth.md
---

# GA4 Data API on Vercel — refresh-token env creds

Getting the GA4 Data API (`@google-analytics/data`) working on Vercel is not obvious, because the two "normal" credential paths both fail here:

- **Service accounts don't work** — GA4's Property/Account Access Management **rejects adding a service-account email** ("This email doesn't match a Google Account"). So an SA, even with a key, never gets property access. (Known GA4 quirk; see [[gcloud-headless-auth.md]] for the fuller saga.)
- **ADC doesn't exist on Vercel** — locally the client reads Application Default Credentials from `~/.config/gcloud/application_default_credentials.json` (an `authorized_user` refresh-token doc). That file isn't on a Vercel function. So the default client auth finds nothing.

## The working pattern

Authenticate as the **property-owning user** via a refresh token, supplied as env vars, and build the client's auth manually:

1. Locally, you already have an `authorized_user` ADC file (minted via the loopback OAuth flow — see [[gcloud-headless-auth.md]]). It contains `client_id`, `client_secret`, `refresh_token`.
2. Set those as Vercel env vars: `GA4_CLIENT_ID`, `GA4_CLIENT_SECRET`, `GA4_REFRESH_TOKEN` (plus `GA4_PROPERTY_ID`).
3. In code, when those env vars are present, construct an OAuth2/`UserRefreshClient` credential and pass it to `BetaAnalyticsDataClient` (via its `auth`/`authClient` option). When they're absent, fall back to ambient ADC so local dev still works unchanged.

Refresh tokens are **not origin-bound**, so the same token that works locally works on Vercel. If GA4 cards start erroring in prod with an auth message, the likely cause is a **revoked/expired refresh token** — re-mint via the loopback flow and update the env var.

## Why a user token, not an SA

It's the same reason ADC was used locally: GA4 only grants property access to real Google accounts, and the owner is a real account. The refresh-token approach just carries that same identity to a serverless environment.

> Established deploying `hdwshopify` to `dashboard.herbariumdyeworks.uk`, 2026-05-26. Pairs with the auth work in [[nextjs-single-user-google-auth]].
