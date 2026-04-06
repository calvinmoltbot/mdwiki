---
title: MarkViewer
tags: [tooling, workflow, documentation]
created: 2026-04-06
updated: 2026-04-06
status: active
related:
  - ../systems/claude-code.md
  - ../systems/mac-mini.md
---

# MarkViewer

MarkViewer is a native macOS markdown reader on Calvin's MacBook. It reads files from an SMB share on the [Mac Mini](mac-mini.md), providing a comfortable reading experience for documents too long for the terminal.

## How It Works

1. Claude Code generates a document (plan, audit, report, brief)
2. Saves it to `/Users/admin/shared/markviewer/<project>/YYYY-MM-DD-description.md`
3. Tells Calvin the filename
4. Calvin opens it in MarkViewer on his MacBook

## When to Use It

- **Over ~20 lines** — plans, audits, analysis, reports, design briefs
- **Tables and formatted content** — MarkViewer renders markdown properly
- **Anything Calvin needs to review carefully** — not a quick terminal glance

Short answers (<20 lines) stay inline in the terminal.

## Folder Structure

```
/Users/admin/shared/markviewer/
├── hermes/          # Hermes agent outputs, cron summaries
│   └── archive/
├── general/         # Cross-project reports, system audits
│   └── archive/
├── golf/            # Golf tournament manager app
├── openrouter-tracker/
├── mdwiki/          # mdwiki project plans and docs
├── roost/
├── home-dashboard/
└── the-bridge/
```

## Conventions

- **Always date-prefix filenames:** `2026-04-06-description.md`
- **Create project folders as needed** — don't dump everything in `general/`
- **Archive old files** to `<project>/archive/` when they're no longer current
- **Tell Calvin the filename** after saving so he can find it

## Relationship to mdwiki

MarkViewer stores **ephemeral output** — point-in-time reports, session summaries, audits. These are sources, not persistent knowledge.

[mdwiki](../index.md) stores **persistent knowledge** — distilled, evolving, cross-referenced. MarkViewer docs feed into wiki pages but the wiki is a separate system.
