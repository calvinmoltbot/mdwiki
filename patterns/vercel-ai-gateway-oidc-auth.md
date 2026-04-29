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

When code runs on a Vercel deployment, Vercel auto-provisions a `VERCEL_OIDC_TOKEN` in the runtime env. The AI SDK's `gateway` provider auto-detects this token and uses it to authenticate.

For local development, `vercel env pull .env.local` downloads a fresh OIDC token (rotates every 12 hours by default).

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
