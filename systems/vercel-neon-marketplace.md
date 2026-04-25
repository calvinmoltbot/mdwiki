---
title: Vercel + Neon (Marketplace pattern)
tags: [vercel, neon, postgres, infrastructure]
created: 2026-04-25
updated: 2026-04-25
status: active
related:
  - ./mac-mini.md
---

# Vercel + Neon Marketplace

Provisioning a Neon Postgres database **through the Vercel Marketplace** is dramatically simpler than the standalone path: no separate Neon API key, no `neonctl auth` browser flow, no env-var copy-paste.

## When to use

Default to this pattern for any new Vercel-hosted project that needs Postgres. The standalone `neonctl` path is only worth it when:

- The DB is shared between multiple non-Vercel runtimes
- You need fine-grained Neon API access (branching scripts, automated RBAC)

For typical Calvin-style projects (one Vercel app, one DB), the marketplace path wins.

## Setup (one-time per project)

1. `vercel link --yes --project <name> --scope <team>` — creates Vercel project, links GitHub
2. Open the project's **Storage** tab in the Vercel dashboard:
   `https://vercel.com/<team>/<project>/stores`
3. **Create Database → Neon (Serverless Postgres)**, name it, pick region
4. Click **Connect** to attach to the project
5. Locally: `vercel env pull .env.local` — pulls **all** Neon env vars

That's it. No Neon CLI, no API key.

## What gets injected

The Vercel-Neon integration adds ~17 env vars to the project. The ones that matter:

| Var | Use |
|-----|-----|
| `DATABASE_URL` | Pooled connection string (use this for app code) |
| `DATABASE_URL_UNPOOLED` | Direct connection (migrations, long transactions) |
| `NEON_PROJECT_ID` | Reference Neon's API if you ever need to |
| `POSTGRES_*`, `PG*` | Various aliases for compatibility with different drivers |

For Drizzle: just `DATABASE_URL` is enough. `drizzle-kit push` reads it from `.env.local`.

## Auth model

The integration is authorized **at the team level** in Vercel. Once the Neon integration is installed for the team, every subsequent project can provision its own DB without re-authorizing. Calvin's `calvin-orrs-projects` team already has it (first installed for the `vinted` project).

## Gotchas

**Branching:** for dev/prod separation, create a Neon branch from the dashboard and the integration creates separate env vars per Vercel environment (development / preview / production). Default behaviour is **one DB serves all envs** — fine for early-stage solo projects, add branches later when there's prod data to protect.

**Service-role secrets:** when the app reads from a *different* Supabase or Neon project (cross-app pattern, e.g. herbarium-hq → herbarium-finance), those secrets still need manual `vercel env add`. The marketplace only auto-wires the DB it provisions.

**`printf` not `echo`:** when piping secrets to `vercel env add`, `echo` appends a `\n` that gets stored as part of the value. Use `printf "%s" "$VALUE" | vercel env add ...`.

## Reference

- Pattern first used: [herbarium-hq](../projects/herbarium-hq.md) (2026-04-25)
