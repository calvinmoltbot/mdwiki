---
title: Supabase CLI / psql access from the Mac Mini
tags: [supabase, postgres, infrastructure, cli, secrets, debugging]
created: 2026-04-26
updated: 2026-04-26
status: active
---

# Supabase CLI / psql access from the Mac Mini

How to run SQL against a Supabase Postgres from CLI on the headless Mac Mini, plus the gotchas that bit us during the herbarium-finance migration on 2026-04-26.

## TL;DR

```bash
export PATH="/opt/homebrew/opt/libpq/bin:$PATH"
PW=$(security find-generic-password -s "<project>-supabase-db" -w)
URL="postgresql://postgres.<project-ref>:$PW@aws-0-<region>.pooler.supabase.com:6543/postgres"
PGPASSWORD="$PW" psql "$URL" -f migration.sql
```

## Pre-reqs

- `libpq` installed via Homebrew (`brew install libpq`). It's keg-only — must export the PATH.
- DB password stashed in macOS Keychain via the `keychain-secret` skill. Naming convention: `<project>-supabase-db`.
- Logged-in SSH session has run `security unlock-keychain` once (otherwise `security` errors with "User interaction is not allowed").

## Pooler ports — port 6543 works, port 5432 may not

Supabase exposes two pooler endpoints on the same host:

| Port | Mode | Use for |
|---|---|---|
| **5432** | Session pooler | Normal connections, prepared statements, `SET` etc |
| **6543** | Transaction pooler | Statement-pooled, scales high-concurrency apps |

**Gotcha (observed 2026-04-26 on herbarium-finance):** the session pooler (5432) rejected the DB password with `error received from server in SCRAM exchange: Wrong password`, but the **same password** worked fine on the transaction pooler (6543). Cause unknown — possibly Supabase regional config, possibly a propagation timing issue after a password reset.

**Implication:** if 5432 says "Wrong password" but you're sure the password is right, try 6543 before assuming the password is wrong. If the migration uses multi-statement transactions (`BEGIN…COMMIT`), be aware that transaction pooler may break those — split into separate files or use the direct DB host.

## The direct DB host doesn't resolve over IPv4

`db.<project-ref>.supabase.co` is **IPv6-only** — `psql` from the Mac Mini fails with `nodename nor servname provided, or not known`. Always go through the pooler.

## Migration files: prefer simple, no `BEGIN/COMMIT`

If you're routing through the transaction pooler, individual statements may go to different backend connections. This breaks multi-statement transactions inside a single SQL file. Prefer migration files that are:

- A sequence of independent DDL/DML statements (each runs as its own implicit transaction)
- Include `\echo` markers + verification SELECTs at the end so you can eyeball the output

## Why not `supabase db push`?

The CLI's `db push` reads `supabase/migrations/*.sql` and tries to apply migrations the local migration tracker thinks are pending on the remote. **Risk:** if the tracker is out of sync (e.g. local has a `000_full_production_schema.sql` dump that's never been pushed but is already the prod state), `db push` will try to re-apply the schema dump and clobber prod.

For one-off migrations, prefer **psql + explicit file** over `db push`. Use `db push` only when you trust the migration tracker state.

## Why not the Supabase Dashboard SQL Editor?

Fine for ad-hoc. Bad for repeatable migrations because there's no record of what was run. Always run from a versioned file in the repo so the migration history is in git.

## Related skills / memory

- Skill: `~/.claude/skills/keychain-secret/SKILL.md` — Keychain conventions and `security` command usage
- Project memory (herbarium-finance): `reference_prod_db_access.md` — concrete Keychain entry name + pooler URL
