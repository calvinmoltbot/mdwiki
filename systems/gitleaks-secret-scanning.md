---
title: Gitleaks Secret Scanning (Global Pre-Commit)
tags: [security, git, tooling, gitleaks]
created: 2026-04-18
updated: 2026-04-18
status: active
related:
  - ../projects/relist.md
---

# Gitleaks Secret Scanning

Three-layer defence against committing secrets to GitHub. Triggered by a Neon `DATABASE_URL` accidentally leaking into `calvinmoltbot/relist` via `git add -A` picking up a `vercel env pull` dump on 2026-04-17. GitHub's native push protection catches OpenRouter/AWS patterns but didn't catch Neon `npg_*`.

## The three layers

1. **`.gitignore`** (per-project) — block `vercel-env*`, `.env.pull*`, `*.secrets`, `secrets.json` before they can be staged.
2. **gitleaks pre-commit hook** (global, all repos) — scans staged content; blocks on match. Active via `git config --global core.hooksPath ~/.githooks`.
3. **GitHub push protection** (per-repo, enabled) — partner program (OpenRouter, AWS, Anthropic, GitHub tokens) auto-revokes on push.

## Global setup

Installed once, applies to every repo on the Mac Mini:

```bash
brew install gitleaks
git config --global core.hooksPath ~/.githooks
# Files: ~/.githooks/pre-commit and ~/.gitleaks.toml
```

Per-repo override: drop a local `.gitleaks.toml` or a local `.githooks/` dir and `git config core.hooksPath .githooks`. Local beats global.

## Custom rules (in `~/.gitleaks.toml`)

Default rules leave gaps. Global config adds:

- **`openrouter-api-key`** — `sk-or-v1-[A-Za-z0-9]{40,}` (matches even without identifier nearby, which `generic-api-key` requires)
- **`neon-postgres-password`** — `postgresql://[^:]+:npg_[A-Za-z0-9]{14,}@` (not in GitHub's native patterns)
- **`anthropic-api-key`** — `sk-ant-(api|admin)NN-...`
- **`env-dump-file`** — catches multi-line `DATABASE_URL=postgres...` dumps specifically by path pattern (`vercel-env*`, `.env.pull*`)

## Hook subcommand — gitleaks v8.30+

`gitleaks protect` was removed in v8.30. Current syntax:

```bash
gitleaks git --staged --redact --verbose        # repo with history
gitleaks dir --redact --verbose <files>          # fresh repo, first commit
```

The hook branches on `git rev-parse --verify HEAD` to handle both cases. Explicit `rc=$?` check after the command — `set -e` doesn't reliably propagate exit codes from inside `if` bodies.

## Bypass (real emergencies only)

```bash
git commit --no-verify
```

## Verification

The `/kickoff` skill runs a silent check at session start and only warns if something is missing (hooksPath unset, gitleaks uninstalled, or `~/.gitleaks.toml` deleted).
