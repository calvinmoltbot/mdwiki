---
title: Single-user Clerk gate via env-var email allowlist
tags: [patterns, auth, clerk, nextjs, vercel]
created: 2026-04-29
updated: 2026-04-29
status: active
sources:
  - mytodo/src/lib/owner-check.ts
  - mytodo/src/middleware.ts
related:
  - ../projects/mytodo.md
---

# Single-user Clerk gate via env-var email allowlist

A pattern for personal/single-user apps that need *real* auth (not just a deployment password) without building a multi-tenant user model.

## Problem

You're building a personal app. Authentication needs to:
- Block randos who find the URL
- Not require a multi-tenant data model (user_id on every row)
- Survive when Clerk lets anyone sign up by default
- Handle the bootstrap problem (you can't lock to your own user before you exist)

## Solution

Three layers:

### 1. Clerk middleware redirects unauthenticated to `/sign-in`

```ts
// src/middleware.ts
export default clerkMiddleware(async (auth, req) => {
  if (!isPublicRoute(req)) {
    const { userId } = await auth();
    if (!userId) {
      const url = new URL("/sign-in", req.url);
      url.searchParams.set("redirect_url", req.nextUrl.pathname + req.nextUrl.search);
      return NextResponse.redirect(url);
    }
  }
});
```

**Don't use `auth.protect()`** on Clerk dev instances — it does a "rewrite" that ends up at /404 when the dev_browser cookie is missing. Explicit `redirectToSignIn` or manual `NextResponse.redirect` is reliable.

**Don't use Clerk's `redirectToSignIn()` either** if you want to use a local sign-in page on a Clerk dev instance — it always routes to Clerk's hosted sign-in. Use `NextResponse.redirect(new URL("/sign-in", req.url))` to force the local page.

### 2. Server-side email allowlist in pages

```ts
// src/lib/owner-check.ts
export function isAllowedUser(user: User | null): boolean {
  if (!user) return false;
  const allowed = (process.env.APP_OWNER_EMAILS ?? "")
    .split(",").map((s) => s.trim().toLowerCase()).filter(Boolean);
  if (allowed.length === 0) return true; // bootstrap: first user wins
  const userEmails = (user.emailAddresses ?? []).map((e) => e.emailAddress.toLowerCase());
  return userEmails.some((email) => allowed.includes(email));
}

// src/app/page.tsx
const user = await currentUser();
if (!isAllowedUser(user)) redirect("/forbidden");
```

### 3. `/forbidden` page with sign-out button

Anyone who signs in but isn't on the allowlist gets a friendly "this is single-user" page with a `<SignOutButton>`. Better UX than a 403.

## Why three layers, not two

- Middleware alone: gates *route access* but Clerk lets anyone sign up. After signup they'd be inside.
- Page check alone: works but lets unauthenticated requests reach page render before redirecting (slower, more code paths).

Together: middleware bounces unauth, page bounces unauth-but-not-the-owner.

## Bootstrap rule

`APP_OWNER_EMAILS` empty → first user wins. Set the env var *immediately* after your own signup. There's a small window where someone else could race you, but for an unannounced URL it's negligible.

If you want zero race risk: pre-set `APP_OWNER_EMAILS` to your email *before* deploying, then sign up.

## Public routes

Don't forget to whitelist:
- `/sign-in(.*)` and `/sign-up(.*)` — auth UI
- `/forbidden` — non-owner landing
- Any webhook endpoints with their own auth (e.g. `/api/telegram` with secret-token + chat-id allowlist)

## Why not a Vercel Deployment Protection password?

It works but:
- Vercel's "Vercel Authentication" doesn't cover custom domains on the free plan
- Password protection is shared (one password for all visitors)
- No per-user identity in your app — can't customize, can't audit, can't show "Hi Calvin"

Clerk gives you real session/user identity for free (up to 10k MAU on the free plan).

## See also

- [mytodo project page](../projects/mytodo.md) — concrete implementation
- Clerk docs: https://clerk.com/docs/quickstarts/nextjs
