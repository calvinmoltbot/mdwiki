---
title: Next.js 16 renamed middleware.ts → proxy.ts
tags: [nextjs, nextjs-16, middleware, proxy, auth, gotcha]
created: 2026-05-26
updated: 2026-05-26
status: active
sources:
  - hdwshopify/src/proxy.ts
related:
  - ../patterns/nextjs-single-user-google-auth.md
  - ./ga4-data-api-on-vercel.md
  - ../decisions/no-turbopack.md
  - ./nextjs-tailscale-dev-origins.md
---

# Next.js 16 renamed middleware.ts → proxy.ts

In Next.js 16, the request interceptor that was `middleware.ts` (`export function middleware`) is now **`proxy.ts`** (`export function proxy`), running on the **Node.js runtime** (not edge). The `config.matcher` export works the same. Internally the build still emits a `middleware.js` artifact, but the **source file convention is `proxy.ts`** — confirmed in the bundled docs at `node_modules/next/dist/docs/01-app/03-api-reference/03-file-conventions/proxy.md`.

If you write `middleware.ts` expecting it to run, it silently won't — the request interceptor never fires.

## Implications for auth

- **Auth.js v5 (next-auth@5) is not confirmed compatible** with this rename at 16.2.6 — its Next integration / edge assumptions predate the proxy change. Rather than fight library-compat on a security boundary, hand-roll auth: a signed (`jose` HS256) httpOnly session cookie + an optimistic check in `proxy.ts` + a real check in the data layer. See [[nextjs-single-user-google-auth]]. The Next 16 auth guide itself recommends this stateless-cookie + proxy-optimistic-check + DAL shape.
- The **proxy is only an optimistic pre-filter** — it does a cookie check and redirects, but the authoritative check belongs in a Data Access Layer (`verifySession()` in a layout) and inside any mutating server action (reachable via direct POST regardless of the proxy).
- Apply security response headers (`X-Frame-Options`, `X-Content-Type-Options`, CSP `frame-ancestors`, `X-Robots-Tag: noindex`) in the proxy too.

## Verifying it actually runs

`next dev` logs the proxy per request: `GET /x 200 in Nms (next.js: …, proxy.ts: …, application-code: …)`. If you don't see a `proxy.ts:` segment, the file isn't wired (wrong name/location).

## Related Next 16 breaking changes

- Turbopack is the default bundler — pass `--webpack` explicitly (see [[no-turbopack]]).
- `revalidateTag(tag, profile)` now takes a required second arg; `updateTag(tag)` is the single-arg, read-your-own-writes variant for server actions.
- Dev over Tailscale needs `allowedDevOrigins` (see [[nextjs-tailscale-dev-origins]]).

> First hit building auth for `hdwshopify` (the Herbarium Dyeworks dashboard), 2026-05-26.
