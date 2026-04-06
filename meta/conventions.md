---
title: Wiki Conventions
tags: [meta, governance]
created: 2026-04-06
updated: 2026-04-06
status: active
related:
  - ../index.md
---

# mdwiki Conventions

This is the wiki's constitution. It defines what belongs, what doesn't, and how pages should be written.

## What mdwiki IS

A **persistent, evolving knowledge base** maintained by Claude Code and Hermes. It captures how things work, why decisions were made, and patterns worth reusing. Calvin reads it via MarkViewer.

## What mdwiki is NOT

- Not a task tracker (use GitHub Issues)
- Not a session log (use MarkViewer for ephemeral reports)
- Not a code reference (read the code itself)
- Not a diary or journal

## What Belongs Here

| Category | Folder | Examples |
|---|---|---|
| **Systems** | `systems/` | How the Mac Mini is set up, how cron jobs run, how deploys work |
| **Projects** | `projects/` | What each project does, architecture, gotchas, current state |
| **Decisions** | `decisions/` | Why we chose X over Y — ADR format |
| **Patterns** | `patterns/` | Reusable solutions, CLI tricks, conventions |
| **Reference** | `reference/` | Account details, service URLs, API keys locations |

## What Does NOT Belong Here

- **Point-in-time reports** — these go to MarkViewer (audits, session summaries, sweep results)
- **Temporary debugging notes** — fix the bug, move on
- **Obvious code documentation** — if the code is self-documenting, don't duplicate it here
- **Speculation or future plans** — only document what IS, not what might be
- **Raw data** — no log dumps, no API responses, no large JSON blobs

## Page Quality Bar

Every page must:

1. **Have complete frontmatter** — title, tags, created, updated, status, related
2. **Be useful standalone** — a reader should understand the page without reading 5 others first
3. **Be distilled, not dumped** — extract the knowledge, don't copy-paste source material
4. **Stay current** — if something changes, update the page. Stale knowledge is worse than no knowledge
5. **Cross-reference related pages** — use `related:` in frontmatter and inline markdown links

## Frontmatter Schema

```yaml
---
title: Page Title
tags: [tag1, tag2]
created: YYYY-MM-DD
updated: YYYY-MM-DD
status: active          # active | stub | needs-review | archived
sources:                # optional — where the knowledge came from
  - markviewer/hermes/2026-04-05-summary.md
  - conversation on 2026-04-03
related:                # other wiki pages
  - systems/mac-mini.md
  - projects/hermes.md
---
```

### Status Values

| Status | Meaning |
|---|---|
| `active` | Current, maintained, trustworthy |
| `stub` | Placeholder — needs content |
| `needs-review` | May be stale or inaccurate — flagged for attention |
| `archived` | No longer relevant but kept for history |

## Writing Style

- **Direct and factual** — no filler, no hedging
- **Present tense** — "Hermes runs cron jobs" not "Hermes was configured to run cron jobs"
- **Imperative for instructions** — "Pass `--webpack` to avoid Turbopack" not "You should pass..."
- **Include the WHY** — don't just say what to do, say why it matters

## Cross-References

Use relative markdown links: `[Mac Mini setup](../systems/mac-mini.md)`

Every page should link to at least one other page. Orphan pages (nothing links to them) get flagged during maintenance sweeps.

## Page Lifecycle

1. **Created** — by Claude Code during a session, by `/wiki add`, or by Hermes
2. **Maintained** — updated when knowledge changes, cross-references added
3. **Reviewed** — weekly maintenance sweep checks for staleness, accuracy, gaps
4. **Archived** — when the subject is no longer relevant (status set to `archived`)

Pages are **never deleted** — only archived. Git history preserves everything.

## Who Writes

- **Claude Code** — primary author. Creates pages during sessions, updates from learnings
- **Hermes** — maintenance. Weekly sweeps fix staleness, broken links, stubs
- **Calvin** — occasional direct edits via any text editor

## Maintenance Rules

The weekly Hermes sweep may autonomously:
- Update stale pages with current information
- Fix broken cross-references
- Flesh out stub pages
- Normalize frontmatter
- Add missing cross-references

The sweep must NOT:
- Rewrite pages from scratch (update, don't replace)
- Create new pages (only flag gaps)
- Delete or archive pages (only flag for Calvin)
- Change the meaning or intent of existing content

All changes are logged in `meta/changelog.md` and git-committed.
