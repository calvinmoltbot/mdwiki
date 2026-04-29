---
title: Vercel AI Gateway authentication via VERCEL_OIDC_TOKEN
tags: [patterns, ai, ai-gateway, vercel, oidc]
created: 2026-04-29
updated: 2026-04-29
status: active
sources:
  - mytodo/src/lib/llm.ts
related:
  - ../projects/mytodo.md
  - ./capture-pipeline-with-llm-routing.md
---

# Vercel AI Gateway auth via OIDC token

Pattern for using Vercel AI Gateway in Next.js apps without managing a separate AI Gateway API key.

## What

For local development, `vercel env pull .env.local` downloads a `VERCEL_OIDC_TOKEN` (rotates every 12 hours). The AI SDK's gateway provider auto-detects it.

**At runtime in production**, `VERCEL_OIDC_TOKEN` is **NOT** automatically available unless OIDC Federation is explicitly enabled at the project level. By default, AI Gateway calls from a deployed Function need either:

1. An explicit `AI_GATEWAY_API_KEY` env var (created via `vercel ai-gateway api-keys create <name>`), OR
2. OIDC Federation enabled — and even then it's intended for federation with external services, not internal AI Gateway calls

**Lesson learned (2026-04-29)**: I assumed the OIDC token would work in production runtime since it works locally. It doesn't. Always set `AI_GATEWAY_API_KEY` for production:

```bash
vercel ai-gateway api-keys create mykey
# copy the vck_... value
printf "%s" "vck_..." | vercel env add AI_GATEWAY_API_KEY production
printf "%s" "vck_..." | vercel env add AI_GATEWAY_API_KEY development
```

Symptom of getting this wrong: `[llm] no auth available — VERCEL_OIDC_TOKEN and AI_GATEWAY_API_KEY both missing` in production logs. Calls silently fall through to whatever fallback you have. Mine was "drop in Inbox without LLM cleanup" — looked exactly like the LLM was returning bad answers.

## Code

```ts
// src/lib/llm.ts
import { generateObject } from "ai";
import { z } from "zod";

export async function cleanupTask(input: ...): Promise<...> {
  if (!process.env.VERCEL_OIDC_TOKEN && !process.env.AI_GATEWAY_API_KEY) {
    return null; // no auth available
  }
  const { object } = await generateObject({
    model: "google/gemini-2.0-flash-lite", // gateway shortcut: provider/model
    schema: SCHEMA,
    prompt: ...,
    temperature: 0.2,
  });
  return object;
}
```

That's it. The AI SDK + `@ai-sdk/gateway` package handle the auth dance.

## Benefits

- **No API key to manage** — no provisioning, no env var rotation, no leaking risk in commits
- **Easy provider swap** — change `"google/gemini-2.0-flash-lite"` to `"openai/gpt-4o-mini"` or `"anthropic/claude-haiku-4-5"` without changing imports or env vars
- **Built-in cost tracking** — Vercel dashboard shows spend per project per provider
- **Failover** — Gateway can fall back across providers (configure in Gateway settings)

## Limits as of 2026-04

- Gateway exposes `languageModel` + `imageModel` only. **No `transcriptionModel`** yet — for Whisper-style audio you still need a direct provider (e.g. Groq).
- Gateway is for inference, not training/fine-tuning.
- Some specialty providers aren't routable through the Gateway — check their availability list.

## Local dev tip

```bash
vercel env pull .env.local
```

Re-run periodically — the OIDC token rotates. If you start seeing 401s from the Gateway in dev, that's the signal.

## When to use

- Any Next.js app on Vercel using AI features
- Multi-provider strategies where you don't want to keep N API keys

## When NOT to use

- Code running outside Vercel — `VERCEL_OIDC_TOKEN` won't be set there. Fall back to a Gateway API key in that case.
- Audio transcription — go direct to provider (e.g. Groq's OpenAI-compatible Whisper endpoint)

## See also

- [mytodo project page](../projects/mytodo.md) — uses this pattern for LLM cleanup
- [Capture pipeline with LLM routing](./capture-pipeline-with-llm-routing.md) — broader context
- Vercel AI Gateway docs: https://vercel.com/docs/ai-gateway
