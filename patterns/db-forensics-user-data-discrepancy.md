---
title: Diagnosing "my data is missing" — DB forensics for user-perceived discrepancies
created: 2026-07-03
updated: 2026-07-03
tags: [debugging, database, postgres, data-integrity, vinted, relist]
related:
  - ../projects/relist.md
---

# Diagnosing user-perceived data discrepancies

When a user swears data is missing/wrong ("I should have 50 listed items, I only see 5"),
resist both jumping to "it's a bug" and "user is mistaken". The DB usually holds the
answer. Pattern used to diagnose Lily's missing ReList listings (2026-07-03).

## Step 1 — Trust nothing, count everything

```sql
SELECT status, count(*) FROM items GROUP BY status ORDER BY 2 DESC;
```

Establish the ground truth in the DB before touching UI/API. Rule out "app not
rendering" (check the live page + API response) separately from "data not present".

## Step 2 — Timestamp forensics reveal *how* rows were created

The killer query: rows where a "later" event predates creation.

```sql
-- sold before it was even created = impossible for organic activity
SELECT count(*) FROM items WHERE status='shipped' AND sold_at < created_at;
```

- `sold_at` at **midnight** (00:00:00) while `created_at`/`shipped_at` carry precise
  system times ⇒ `sold_at` is a **user-entered date**, not a system event.
- `sold_at < created_at` in bulk ⇒ user is **retroactively logging past sales**, not
  tracking live inventory. This reframed Lily's whole usage: ReList was a *sales ledger*,
  not a *live listing tracker*. Her "missing" listings were simply never entered.

- Bulk `updated_at`/`shipped_at` clusters (dozens sharing an exact minute) ⇒ batch
  "mark as shipped" actions, not per-item events.

## Step 3 — Cross-reference against the source of truth

The DB can only tell you what it *has*. To prove items are genuinely absent (vs
mis-statused), compare against the external system. For Vinted:

1. Get the shop's member id from any item page:
   `document.querySelectorAll('a[href*="/member/"]')`
2. Live listings render in the profile grid but the grid **lazy-loads** — programmatic
   `window.scrollTo` often won't trigger it; use the `computer` tool's real mouse-wheel
   scroll to the footer, then harvest `a[href*="/items/"]` ids.
   (Vinted's `/api/v2/users/:id/items` query-string endpoint was 404/guarded — DOM harvest
   was the reliable path. Public grid may cap below the header count; reserved/hidden
   items don't render.)
3. Cross-join the harvested ids against the DB by **both** external URL and normalised name:

```sql
WITH live(vid) AS (VALUES ('9310899132'), ...)
SELECT l.vid, i.sku, i.status
FROM live l LEFT JOIN items i ON i.vinted_url ILIKE '%'||l.vid||'%';
```

`NULL` sku on the join = genuinely absent (never persisted), **not** mis-statused.
That's what closed the case: the older listings weren't wrongly flipped to sold — they
were never sent to the app.

## Takeaways

- **Midnight timestamps = human input; precise timestamps = system events.** Fastest tell
  for "imported/backfilled" vs "organic" rows.
- **Absent ≠ mislabeled.** A LEFT JOIN against the external source distinguishes them
  cheaply and ends the "is it a bug or the user?" debate with evidence.
- **Back up before any remediation** (`pg_dump --no-owner --no-acl`; plain `pg_dump` can
  exceed a 2-min tool timeout on Neon — background it or use flags).
- Templated/duplicate item names are a latent hazard for any name-based dedup — see the
  companion fix in [relist](../projects/relist.md) (PR #55).
