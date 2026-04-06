---
title: Claude Code Workflow
tags: [tooling, workflow, skills]
created: 2026-04-06
updated: 2026-04-06
status: active
related:
  - ../systems/mac-mini.md
  - ../systems/markviewer.md
  - ../systems/cron-pipeline.md
  - ../projects/hermes.md
---

# Claude Code Workflow

Claude Code is the primary development tool. It runs on the [Mac Mini](mac-mini.md) and is how most code gets written, reviewed, and shipped.

## Session Rituals

The standard workflow loop:

1. **`/kickoff`** — Start a session. Shows git state, open issues, dirty files, handoffs. Names the iTerm2 tab via tmux. Helps pick an issue and branch off main.
2. **Work on branch** — Implement the feature or fix.
3. **`/wrap-up`** — End the session. Commits work, creates issues for loose ends, opens a PR if on a feature branch.
4. **`/workbench`** — Bird's-eye view across all projects. Git status, issues, PRs, active sessions, dev servers.

The loop: `/kickoff` → pick issue → work → `/wrap-up` → `/workbench` → next project.

## Skills

Skills live in `~/.claude/skills/` and extend Claude Code with specialized capabilities. Key skills:

| Skill | Purpose |
|---|---|
| `kickoff` | Session start ritual |
| `wrap-up` | Session end ritual |
| `workbench` | Cross-project dashboard |
| `workflow` | GitHub dev workflow (issues → branches → PRs) |
| `frontend-design` | Production-grade UI generation |
| `playground` | Interactive HTML explorers |
| `recursive-build` | Parallel agent workflow for multi-feature builds |
| `health-check` | Mac Mini diagnostics and cleanup |
| `export-secrets` | Export credentials to Secrets Vault |
| `agent-handoff` | Process handoffs between Claude and Hermes |
| `pdf`, `docx`, `pptx`, `xlsx` | Office document handlers |
| `canvas-design` | Visual art in PNG/PDF |
| `skill-creator` | Create new skills |
| `socratic` | Reasoning enhancer |

## Memory System

Claude Code has persistent file-based memory at `~/.claude/projects/<project>/memory/`. Memories are markdown files with frontmatter (name, description, type). Types: `user`, `feedback`, `project`, `reference`.

`MEMORY.md` is an index loaded into every conversation — keep it concise (<200 lines).

## MarkViewer Integration

Any output over ~20 lines goes to [MarkViewer](markviewer.md), not the terminal:
- Save to `/Users/admin/shared/markviewer/<project>/YYYY-MM-DD-description.md`
- Always date-prefix filenames
- Tell Calvin the filename after saving
- Short answers (<20 lines) stay inline

## GitHub Integration

- Every idea, bug, or TODO should become a GitHub issue
- Standard labels across all repos: `bug`, `design`, `ready-for-claude`, `for-hermes`
- Labels must be created before use: `gh label create "name" --repo OWNER/REPO --color "HEX" --force 2>/dev/null`
- Handoff repo: `calvinmoltbot/handoffs` with `for-claude` label for pending tasks

## Env Var Convention

When piping values to `vercel env add`, use `printf` not `echo`. Echo appends a trailing `\n` that gets stored as part of the value, causing silent failures.
