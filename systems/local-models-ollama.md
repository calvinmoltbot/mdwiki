---
title: Local Models on the Mac Mini (Ollama)
tags: [ai, local-llm, ollama, mac-mini, huggingface]
created: 2026-06-22
updated: 2026-06-22
status: active
related:
  - ../systems/mac-mini.md
  - ../systems/nextjs-tailscale-dev-origins.md
---

# Local Models on the Mac Mini (Ollama)

How Calvin runs and tests open-weight LLMs locally on the [Mac Mini](mac-mini.md) (Apple M4, 16GB) via Ollama, driven from the MacBook over Tailscale. Grew into the **model-lab** project (`calvinmoltbot/model-lab`).

## The hardware ceiling (the constraint everything bends around)

- M4 / **16GB unified RAM**. macOS takes ~3–4GB, leaving ~12GB for model + context.
- Practical ceiling ≈ **12B at Q4** (~7–8GB weights). It *fits* but is tight — expect ~1GB swap and ~13 tok/s.
- **7–9B is the comfortable sweet spot** (~5–6GB).
- **26B+ won't fit** — swaps to disk and crawls.
- Advertised long context windows (128K–256K) are theoretical here: the weights fit, a large KV cache does not. Keep context modest.
- A loaded model is the real RAM cost; the idle `ollama serve` process with nothing loaded is cheap.

## The on/off pattern (avoid idle resource drain)

The **GUI `Ollama.app` is the wrong setup for a headless Mini**: it registers an SMAppService login item (`com.ollama.ollama`) that auto-starts at boot, binds **localhost-only** (so the MacBook can't reach it), and keeps a server resident. On a headless box the menubar UI is invisible anyway — pure cost.

Replacement: a **controllable headless launchd agent** with explicit on/off.

- Disable the GUI autostart: `launchctl bootout gui/$(id -u)/com.ollama.ollama` + `launchctl disable …`.
- Agent `~/Library/LaunchAgents/com.calvin.ollama.plist`: `RunAtLoad=false`, `KeepAlive=false` (never boots, never auto-respawns — explicit control only). Env:
  - `OLLAMA_HOST=0.0.0.0:11434` — bind all interfaces so the MacBook reaches it over Tailscale (`http://100.90.11.37:11434`).
  - `OLLAMA_KEEP_ALIVE=5m` — unload a model from RAM 5 min after last use (frees ~7–8GB).
  - `OLLAMA_MAX_LOADED_MODELS=1` — only one model resident at a time on 16GB.
- Shell helpers in `~/.zshrc`: **`ollama-on`** (kickstart agent + the chat UI, print URLs), **`ollama-off`** (kill process → frees RAM instantly), **`ollama-status`** (what's up + what's resident via `/api/ps`).

Net effect: zero idle footprint when not dabbling; one command to bring it up bound for the MacBook.

## Reaching it from the MacBook

Ollama bound `0.0.0.0` → MacBook hits the Tailscale IP. Also OpenAI-compatible at `…:11434/v1` (dummy API key). For a browser chat UI, a **zero-dependency stdlib Python server** (`~/.ollama/chat-ui/`) serves a static page and reverse-proxies `/api/*` to Ollama — **same-origin, so no CORS config needed**. Chosen over Open WebUI (a ~1GB install + persistent DB/auth service) because it's lean and on-demand — fits the "don't hog resources" goal.

## Running Hugging Face GGUFs directly

Ollama can run any HF GGUF without a manual download/Modelfile:

```
ollama run hf.co/<user>/<repo>:<QUANT>      # e.g. :Q4_K_M
```

To check a model's real size *before* pulling (cheap — metadata only, no blob download):

- **Ollama registry manifest:** `GET https://registry.ollama.ai/v2/library/<model>/manifests/<tag>` with `Accept: application/vnd.docker.distribution.manifest.v2+json` → sum the layer sizes. (MLX-variant tags use a non-Docker mediaType and return 412 to this probe.)
- **HF repo file tree:** `GET https://huggingface.co/api/models/<repo>/tree/main?recursive=true` → per-file sizes; filter `.gguf`.
- **HF search (trending):** `GET https://huggingface.co/api/models?search=<q>&sort=trendingScore` → public JSON, more reliable than scraping the web UI.

Don't trust `WebFetch` summaries for exact sizes/tags — its small summarizer model fabricates clean-looking tables. Hit the JSON APIs above for ground truth.

## Gotchas

- **The `thinking` field vs token leak.** Reasoning-tuned fine-tunes (e.g. Gemma-4 agentic distills) emit `<|channel>thought…<channel|>` inline on the **raw `/api/generate`** endpoint, but `/api/chat` cleanly separates reasoning into a separate `"thinking"` field — `"content"` stays clean. Use `/api/chat` and read/hide `thinking` separately.
- **Community fine-tune provenance.** The most "trending" HF models are often anonymous distillations (e.g. trained on Claude/Fable-5 outputs). Fine for dabbling; don't build anything load-bearing on them without your own evals.
- **Killing a respawning service ≠ stopping it.** If a port refuses to free after `kill -9` and the parent PID is 1, it's a launchd-managed service (parent process), not a stale dev server — stop it with `launchctl bootout`/`disable`, not `kill`. (Diagnosed when letterboxd-tracker's `next start` on :3040 kept respawning.)

## See also

- [Mac Mini](mac-mini.md) — host, Tailscale IPs, ports
- [Next.js + Tailscale dev origins](nextjs-tailscale-dev-origins.md) — same cross-device binding gotchas for the model-lab dashboard
