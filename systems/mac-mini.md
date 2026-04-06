---
title: Mac Mini Setup
tags: [infrastructure, networking, hardware]
created: 2026-04-06
updated: 2026-04-06
status: active
related:
  - ../systems/claude-code.md
  - ../systems/markviewer.md
  - ../reference/accounts-and-services.md
---

# Mac Mini Setup

Calvin's headless Mac Mini is the development server. Calvin SSHs into it from his MacBook. Claude Code, Hermes, and all dev processes run here. The MacBook is for browsing, reading MarkViewer docs, and Chrome.

## Network Access

- **Tailscale IP:** `100.90.11.37`
- Calvin connects via SSH over Tailscale
- Dev servers on the Mini are accessed from the MacBook at `http://100.90.11.37:<port>`

## Dev Server Conventions

All dev servers must bind to all interfaces so the MacBook can reach them:

| Framework | Command |
|---|---|
| Next.js | `next dev --hostname 0.0.0.0 --webpack` |
| Vite | `vite --host 0.0.0.0` |
| Python | `python3 -m http.server <port> --bind 0.0.0.0` |

**Turbopack is broken for remote access.** Next.js 16 made Turbopack the default bundler, but it has a bug where non-localhost connections hang (TCP establishes but no HTTP response). Always pass `--webpack` explicitly. See [No Turbopack decision](../decisions/no-turbopack.md).

## Clipboard

`pbcopy` doesn't work on headless Mac Mini. Clipboard access over SSH requires **OSC 52** escape sequences.

## Screenshots

Calvin takes screenshots on the MacBook using Lightshot and saves them (Cmd+S) to a shared SMB folder. They arrive on the Mini at `~/screenshots/`.

To check recent screenshots:
```bash
ls -lt ~/screenshots/ | head -5
```

The Read tool can view PNG/JPG images directly.

## Serving Local Files

`open file.html` only opens on the headless Mini — Calvin can't see it. Instead, spin up a temporary HTTP server:

```bash
python3 -m http.server <port> --bind 0.0.0.0 &
```

Then tell Calvin to visit `http://100.90.11.37:<port>/<filename>`. Kill when done:

```bash
pkill -f "python3 -m http.server <port>"
```

This applies to playgrounds, canvas-design output, and any HTML artifacts.

## SMB Shares

The Mini shares folders accessible from the MacBook:
- `/Users/admin/shared/markviewer/` — MarkViewer document store
- `/Users/admin/shared/mdwiki/` — This wiki
