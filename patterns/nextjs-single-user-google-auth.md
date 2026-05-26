---
title: Single-user Google auth for Next.js 16 (hand-rolled)
tags: [patterns, auth, nextjs, nextjs-16, google-oauth, jose, vercel]
created: 2026-05-26
updated: 2026-05-26
status: active
sources:
  - hdwshopify/src/lib/auth/config.ts
  - hdwshopify/src/lib/auth/session.ts
  - hdwshopify/src/lib/auth/dal.ts
  - hdwshopify/src/proxy.ts
related:
  - ./single-user-clerk-gate.md
  - ../systems/nextjs-16-proxy-rename.md
  - ../systems/ga4-data-api-on-vercel.md
---

# Single-user Google auth for Next.js 16 (hand-rolled)

A real-identity auth gate for a personal/single-user app on **Next.js 16**, with **no auth library**. Use when the app exposes private data or can mutate production (e.g. a store-management tool) and a shared deployment password isn't enough.

This is the **library-free sibling** of [[single-user-clerk-gate]] — choose this when you're on Next 16 (where Auth.js v5 compatibility is unconfirmed, see [[nextjs-16-proxy-rename]]) and want zero third-party auth dependency.

## Shape

Google OAuth (web client) → a signed (`jose` HS256) httpOnly session cookie carrying the email → three enforcement layers, all checking a single allow-listed email (`ALLOWED_EMAIL` env):

1. **`proxy.ts`** — optimistic cookie check, redirects unauthenticated requests to `/signin`; also sets security headers. Cookie-only, no network calls. (Proxy, not middleware — Next 16, see [[nextjs-16-proxy-rename]].)
2. **DAL `verifySession()`** — the real check, called in the `(app)` layout so it covers every page; redirects on failure. Memoise with React `cache`.
3. **Server-action guard** — any mutating action re-checks `isAuthorized()` itself, because actions are reachable via direct POST regardless of the proxy. Reject before mutating.

## Fail-closed rule (critical)

`authEnforced() = isAuthConfigured() || isProductionRuntime()`. So if the auth env vars are missing **in production**, the app still enforces (and the cookie verify fails → deny), rather than serving data open. `decrypt()` swallows a missing-`AUTH_SECRET` error and returns null, so an unconfigured prod never accidentally authorises. Local dev runs **open** when auth env is absent, so development isn't blocked.

## Env vars

`AUTH_SECRET` (`openssl rand -base64 32`), `AUTH_GOOGLE_ID`, `AUTH_GOOGLE_SECRET`, `ALLOWED_EMAIL`, `AUTH_URL` (the canonical https origin, so the OAuth redirect URI always matches). OAuth client is a **Web application** type with redirect `https://<domain>/api/auth/callback` (+ `http://localhost:3000/...` for dev); consent screen in Testing with the owner as a test user is enough (openid/email/profile are non-sensitive, no verification needed).

## Secret-handling tip (headless Mini + MacBook browser)

When the OAuth client secret only displays in the browser (MacBook) but the CLI runs on the Mini, the cleanest way to set it without it touching chat or shell history: **copy the secret in-page → paste straight into the Vercel dashboard env-var form**. Non-secret values (client ID, property ID) and file-sourced secrets can still go via `printf '%s' … | vercel env add` (never `echo` — trailing newline).

> Built for `hdwshopify` (Herbarium Dyeworks dashboard) deploy, 2026-05-26. GA4 data in the same app authenticates separately — see [[ga4-data-api-on-vercel]].
