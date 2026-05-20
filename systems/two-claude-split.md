---
title: Two-Claude split (Mini + MacBook)
tags: [workflow, claude-code, tailscale, mini, macbook]
created: 2026-05-16
updated: 2026-05-16
status: active
related:
  - mac-mini.md
  - ../projects/letterboxd-tracker.md
---

# Two-Claude split (Mini + MacBook)

When work spans both machines — code on the Mini, but daemons / VPN / hardware on the MacBook — run two separate Claude sessions and split ownership. Avoids the "Mini-Claude can't see Surfshark, MacBook-Claude can't see the repo" deadlock.

## Why two sessions, not one

- Mini is headless and runs the dev servers, launchd agents, Plex, the repo working tree, and the Vercel deploy origin.
- MacBook runs Transmission (bound to Surfshark `utun5`), Surfshark itself, screenshots, the browser Claude uses for UI tests, and anything that needs a real desktop session.
- Cross-machine SSH is one-shot only — fine for `defaults read`, painful for editing files or watching a service.

## Ownership map

| Concern | Owner | Notes |
|---|---|---|
| Repo working tree | **Mini-Claude** | `/Users/admin/Dev/Projects/...` is canonical |
| `pnpm dev` / `pnpm build` / `mini-rebuild.sh` | **Mini-Claude** | Service runs on Mini |
| launchd agents (`com.calvin.*`) | **Mini-Claude** | Mini-side daemons |
| Vercel deploys | **Mini-Claude** | `vc` CLI is wired up on Mini |
| Plex server / `plex-sync` | **Mini-Claude** | Plex runs on Mini |
| Transmission daemon | **MacBook-Claude** | `~/Library/Application Support/Transmission` |
| `script-torrent-done` hook | **MacBook-Claude** | Lives at `~/Library/.../scripts` on MacBook |
| Surfshark VPN / WireGuard | **MacBook-Claude** | `utun5`, `wg-quick`, kill-switch |
| MacBook LaunchAgents | **MacBook-Claude** | e.g. Transmission auto-start |
| GitHub issues / PRs | Either | Both can `gh` against the same repo |

## How they communicate

- **GitHub issues** — the durable handoff channel. Label issues `for-macbook` or `for-mini` if the scope isn't obvious from the title.
- **Markviewer notes** in `/Users/admin/shared/markviewer/<project>/` are on the Mini's filesystem and **not** visible to MacBook-Claude unless SMB-mounted. Use issues, not notes, for cross-machine handoffs.
- **No direct cross-edits.** MacBook-Claude should not edit files in the Mini's repo over SSH/SMB. If a repo change is needed, open a PR from MacBook (clone separately, or describe the change in an issue for Mini-Claude to implement).

## When MacBook-Claude needs to touch the repo

Two valid patterns:

1. **Describe, don't edit.** File a GitHub issue describing the needed code change. Mini-Claude picks it up on next `/kickoff`. Best for anything bigger than a one-liner.
2. **Clone locally into `~/Dev/`.** The MacBook already has a `~/Dev/` folder from Calvin's pre-Mini coding days — use it. `gh repo clone calvinmoltbot/<repo> ~/Dev/<repo>`, branch, push, PR. Two checkouts of the same repo is fine as long as both push to GitHub and neither tries to `git pull` over Tailscale from the other.

Never edit the Mini's working tree from the MacBook (SMB or SSH). That breaks Mini-Claude's mental model of git state and leads to "uncommitted changes I didn't make" confusion at next kickoff.

## Where to launch MacBook-Claude

- **System diagnostics** (Transmission, Surfshark, `defaults`, `ifconfig`) → anywhere; `~` is fine.
- **MacBook-local scripts** (e.g. `script-torrent-done` hook in `~/Library/Application Support/Transmission/scripts/`) → launch from that folder so Claude has direct file access.
- **Repo code changes** → launch from inside `~/Dev/<repo>` after cloning. Don't reach into the Mini's checkout over SMB or SSH.

## Diagnostic patterns

**MacBook-Claude — Transmission/Surfshark verification one-liner:**

```bash
echo "=== Transmission bind ===" && \
defaults read org.m0k.transmission bind-address-ipv4 2>/dev/null && \
echo "=== utun5 (Surfshark) ===" && \
ifconfig utun5 2>/dev/null | grep -E 'inet|status' && \
echo "=== Public IP via Transmission's interface ===" && \
curl --interface utun5 -s https://ifconfig.me && echo
```

**Mini-Claude — check MacBook reachability:**

```bash
# Raw IP works regardless of MagicDNS state
nc -zv 100.115.176.116 9091
```

## Known cross-machine gotchas

- **MagicDNS is off on the Mini** (`tailscale set --accept-dns=false` at some point). The MacBook's `calvins-macbook-air-3.tail9422ba.ts.net` hostname does **not** resolve from the Mini. Use the raw Tailscale IP (`100.115.176.116`) in any Mini-side config that points at the MacBook.
- **Mini's `pbcopy` is dead** (headless). Use OSC 52 escape sequences if anything needs to round-trip a string to Calvin's MacBook clipboard.
- **Screenshots** land on the Mini at `~/screenshots/` via SMB share. MacBook-Claude does not see the same path; Calvin's MacBook stores Lightshot output locally before the SMB save.
