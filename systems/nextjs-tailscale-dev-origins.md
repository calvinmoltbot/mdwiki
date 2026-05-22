---
title: Next.js 16 dev server won't hydrate over Tailscale (allowedDevOrigins)
tags: [nextjs, tailscale, hydration, dev-server, debugging, mac-mini]
created: 2026-05-22
updated: 2026-05-22
status: active
related:
  - ./mac-mini.md
---

# Next.js 16 dev server won't hydrate over Tailscale (allowedDevOrigins)

**Symptom**: a Next.js 16 dev server bound to `0.0.0.0` and accessed over Tailscale
(e.g. `http://100.90.11.37:<port>` — the Mini's Tailscale IP) renders the page, but it
is **completely inert in the browser**:

- Buttons / links do nothing (no client navigation)
- Client-only components render blank (e.g. Recharts charts — empty containers)
- No console errors; React's "Download React DevTools" message *does* appear
- DOM nodes have **no `__reactFiber$…` / `__reactProps$…` keys** → React never hydrated
- `curl` of the same URL looks **perfect** (server-rendered HTML is fine)

Looks like a chart/CSS bug or a client crash. **It isn't.** The page never hydrated.

## Cause

Next 16 blocks **cross-origin** dev-server requests (the RSC / HMR payloads the client
needs to hydrate) unless the origin is listed in `allowedDevOrigins`. The browser is on
the MacBook hitting the Mini's Tailscale IP — a **different origin** from `localhost` —
so those requests are rejected and hydration data never arrives. The dev server prints a
one-line `allowedDevOrigins` warning + doc link at startup; that warning is the tell.

## Fix

Add the Tailscale IP to `allowedDevOrigins` in `next.config.ts` (dev only):

```ts
// next.config.ts
const nextConfig: NextConfig = {
  ...(process.env.NODE_ENV === "development"
    ? { allowedDevOrigins: ["100.90.11.37"] } // Mini's Tailscale IP
    : {}),
};
```

Restart the dev server (config changes aren't hot-reloaded).

## Why it's easy to miss

`curl` only exercises server-rendered output and passes even when the browser page is
dead. **Validate UI/client-interactive work in a real browser** (Claude-in-Chrome over
the Tailscale URL): screenshot, check the console, and confirm hydration (`__reactFiber$`
keys present). Applies to **every** Next 16 project viewed over Tailscale — pair this
with `--hostname 0.0.0.0 --webpack` (see [[mac-mini]] dev-server notes).

## Related trap

In the same debugging session, Recharts `<ResponsiveContainer>` measured `width:0` on its
first mount inside a streamed `<Suspense>` subtree and never recovered (chart blank until
a client navigation). Fix: self-measure the container with a `ResizeObserver` and render
the chart with an explicit numeric width once known, instead of `<ResponsiveContainer>`.
