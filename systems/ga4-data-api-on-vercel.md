---
title: GA4 Data API on Vercel — refresh-token env creds
tags: [ga4, google-analytics, vercel, auth, adc, oauth, gotcha]
created: 2026-05-26
updated: 2026-05-31
status: active
sources:
  - hdwshopify/src/lib/ga4.ts
related:
  - ./gcloud-headless-auth.md
  - ./vercel-deploy-gotchas.md
  - ../patterns/nextjs-single-user-google-auth.md
  - ../projects/hdwshopify.md
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

## ⚠️ The 7-day token death (general Google-OAuth gotcha)

If the refresh token starts failing with `invalid_grant` ("Token has been expired
or revoked") roughly a **week after it was minted**, the root cause is almost
always that the OAuth **consent screen is in "Testing" publishing status** — Google
caps refresh tokens at 7 days for Testing-status apps. This is **not GA4-specific**;
it hits any Google OAuth integration (Gmail, Calendar, Drive, …) using a stored
refresh token.

**Permanent fix:** GCP console → Google Auth Platform → **Audience → Publish app →
"In production"** (`console.cloud.google.com/auth/audience?project=<PROJECT>`). For
an unverified sensitive scope (e.g. `analytics.readonly`), consent then shows a
one-time "Google hasn't verified this app" warning (Advanced → continue) — harmless
when you own the client secret. In-production tokens don't hit the 7-day cap; don't
click "Back to testing".

**Two credential stores — keep in sync.** A re-mint must update **both** the local
ADC file (`~/.config/gcloud/application_default_credentials.json`) **and** the
Vercel env var `GA4_REFRESH_TOKEN` — they drift independently. After updating the
env var, **redeploy** (`vercel redeploy <prod-url>`); env changes only apply to new
deployments. `client_id`/`client_secret` are unchanged across re-mints (same OAuth
client). First diagnosed on hdwshopify 2026-05-31 — see [[hdwshopify]].

## Why a user token, not an SA

It's the same reason ADC was used locally: GA4 only grants property access to real Google accounts, and the owner is a real account. The refresh-token approach just carries that same identity to a serverless environment.

> Established deploying `hdwshopify` to `dashboard.herbariumdyeworks.uk`, 2026-05-26. Pairs with the auth work in [[nextjs-single-user-google-auth]].
