---
title: Agent Handoff Workflow
tags: [patterns, agents, workflow, hermes]
created: 2026-04-06
updated: 2026-04-06
status: active
sources:
  - markviewer/hermes/2026-04-05-handoff-workflow-proposal.md
related:
  - ../projects/hermes.md
  - ../systems/claude-code.md
---

# Agent Handoff Workflow

How Claude Code and Hermes coordinate work via GitHub issues.

## Repo

`calvinmoltbot/handoffs` — dedicated repo, no code, just issues.

## Labels (5 total)

| Label | Meaning |
|---|---|
| `for-claude` | Claude Code should pick this up |
| `for-hermes` | Hermes should pick this up |
| `in-progress` | Being worked on |
| `done` | Completed |
| `blocked` | Needs Calvin's input |

## Ownership Boundaries

| Agent | Owns |
|---|---|
| **Claude Code** | All coding, architecture, PRs, infrastructure, deployments |
| **Hermes** | Discovery sweeps, Telegram comms, cron jobs, smart home (future), email triage (future) |
| **Shared** | The Bridge — Claude builds code, Hermes feeds data |

## Workflow

1. Agent creates issue with `for-claude` or `for-hermes` label
2. Adds context in the issue body (What, Context, From, Expected Result)
3. Notifies Calvin via Telegram or session start (`/kickoff` checks for handoffs)
4. Receiving agent: comment "picking this up" → add `in-progress` → do work → comment results → add `done` → close

Closed issues are the archive — no separate archiving step needed.

## Issue Template

```markdown
## What
[What needs to happen]

## Context
[Why this is needed, what prompted it]

## From
[Which agent/person created this]

## Expected Result
[What done looks like]
```
