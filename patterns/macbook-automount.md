---
title: MacBook SMB Automount
tags: [patterns, infrastructure, macos]
created: 2026-04-06
updated: 2026-04-06
status: active
sources:
  - markviewer/general/archive/2026-04-03-macbook-setup.md
related:
  - ../systems/mac-mini.md
---

# MacBook SMB Automount

A LaunchAgent pattern for automatically mounting Mac Mini SMB shares when the MacBook boots.

## Config File

`~/Library/LaunchAgents/com.calvin.mount-mini-shares.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.calvin.mount-mini-shares</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>
sleep 10
open "smb://guest:@192.168.7.30/dev"
sleep 2
open "smb://guest:@192.168.7.30/screenshots"
    </string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>StandardErrorPath</key>
  <string>/tmp/mount-mini-shares.err</string>
  <key>StandardOutPath</key>
  <string>/tmp/mount-mini-shares.log</string>
</dict>
</plist>
```

## Commands

| Action | Command |
|---|---|
| Load | `launchctl load ~/Library/LaunchAgents/com.calvin.mount-mini-shares.plist` |
| Test | `launchctl start com.calvin.mount-mini-shares` |
| Unload | `launchctl unload ~/Library/LaunchAgents/com.calvin.mount-mini-shares.plist` |
| Check errors | `cat /tmp/mount-mini-shares.err` |

## Result

Shares mount at `/Volumes/dev` and `/Volumes/screenshots`. Can be dragged to Finder Favourites for quick access.
