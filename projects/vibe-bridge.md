# Vibe Bridge

**Repo:** [calvinmoltbot/vibe-bridge](https://github.com/calvinmoltbot/vibe-bridge)
**Status:** Paused — daemon disabled, not running
**Stack:** Python, WebSocket, SwiftBar

## What it does

Local-first monitoring bridge for remote AI coding sessions over SSH. Shows active Claude Code sessions in the macOS menu bar via a SwiftBar plugin.

- **Mac Mini:** runs a WebSocket daemon (`bridge.daemon`) that captures Claude Code hook events
- **MacBook:** runs a receiver + SSH tunnel, writes state to a local JSON file
- **SwiftBar plugin** (`local/vibebridge.1s.py`) renders session status in the menu bar

## Architecture

```
Claude Code hooks → claude_hook.py → daemon (Mini :8765)
    → WebSocket → SSH tunnel → receiver (MacBook :9876) → state.json → SwiftBar
```

## Why it's paused

The core limitation is that it doesn't work well over SSH — the hook-to-daemon-to-tunnel-to-receiver chain is fragile and hard to keep alive. Calvin wants to think of a better approach before investing more time.

## Files on disk

- **Repo:** `/Users/admin/Dev/Projects/vibe-bridge/`
- **Runtime dir:** `~/.vibe-bridge/` (daemon.log, hooks/claude_hook.py)
- **Launchd plist:** `~/Library/LaunchAgents/com.vibebridge.daemon.plist` (disabled)
- **Configs:** `config.mini.json` (Mini), `config.macbook.json` (MacBook)

## To reactivate

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.vibebridge.daemon.plist
launchctl kickstart gui/$(id -u)/com.vibebridge.daemon
```
