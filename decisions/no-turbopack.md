---
title: "ADR: No Turbopack"
tags: [decision, nextjs, infrastructure]
created: 2026-04-06
updated: 2026-04-06
status: active
related:
  - ../systems/mac-mini.md
---

# Decision: Always Use --webpack, Never Turbopack

## Context

Next.js 16 made Turbopack the default bundler, replacing Webpack.

## Problem

Turbopack has a bug where non-localhost connections hang. TCP connection establishes but no HTTP response is ever sent. This is a showstopper for our setup where the dev server runs on the [Mac Mini](../systems/mac-mini.md) (headless) and is accessed from the MacBook via Tailscale at `http://100.90.11.37:<port>`.

## Decision

Always pass `--webpack` explicitly when running Next.js dev servers:

```bash
next dev --hostname 0.0.0.0 --webpack
```

## Consequences

- Slightly slower HMR than Turbopack would provide
- Must remember to add the flag on every project
- This decision should be revisited when Turbopack fixes the non-localhost bug
