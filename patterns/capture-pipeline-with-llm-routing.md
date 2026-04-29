---
title: Capture pipeline with inline LLM routing
tags: [patterns, ai, llm, ai-gateway, capture, routing]
created: 2026-04-29
updated: 2026-04-29
status: active
sources:
  - mytodo/src/lib/capture.ts
related:
  - ../projects/mytodo.md
---

# Capture pipeline with inline LLM routing

A pattern for ingesting user content from heterogeneous sources (chat bot, web, voice) into a structured store with the right *destination* picked at insert-time by a cheap LLM.

## Problem

User has multiple "buckets" (lists, projects, categories) and wants to drop captured content into the right one without manual tagging every time. Naïve approach: drop everything in an Inbox + show suggestion chips for the user to triage later. Works, but adds a manual step per item.

## Solution

A single `prepareCapture(raw, source, opts)` helper that runs *before* insert and resolves four fields:

1. **`listId`** — destination
2. **`content`** — final stored text (cleaned for voice, verbatim for text)
3. **`transcription`** — raw original (voice only)
4. **`suggestedTags` / `dueAt` / `recurrence`** — extracted side-data

### Routing precedence

```
1. Explicit name:-prefix in user text  →  use that list
2. LLM picks from list of existing lists  →  use that
3. Per-chat / per-source default  →  use that
4. Global Inbox  →  fallback
```

### Why precedence matters

- **Prefix wins** because the user explicitly opted into routing — never override their literal command
- **LLM next** because in single-user personal apps the LLM is right ~95% of the time and a wrong call is cheap to undo (one-click move-task action)
- **Per-source default** for chat bots where the user has set a preference (Telegram `/list garden` persists in a `chat_settings` table)
- **Inbox** is the safe fallback — never silently lose a capture

### Voice vs text behavior

- **Voice**: apply LLM-cleaned title as the content (drop "umm so I need to remember to" filler), preserve raw transcription in a separate column
- **Text**: keep verbatim. Surface cleaned title as a chip suggestion the user can accept

This matches user intent: voice transcription noise is unwanted; typed text is intentional.

## Implementation notes

- **Run synchronously** before insert. Latency cost is ~200-800ms with Gemini Flash Lite. Acceptable for a Telegram bot reply because the alternative ("dropped in Inbox, ✨ suggestion appears later") felt naff to the user.
- **Use AI SDK `generateObject` with a Zod schema** so suggestions come back typed and validated. No JSON parsing edge cases.
- **Failures fall through silently** — wrap LLM call in `.catch(() => null)`. If the model is down or the gateway hiccups, fall back to Inbox + raw content. Don't fail the capture.
- **Bot reply confirms destination** — say "✅ added to **Garden** (auto-routed)" so the user knows the LLM picked it. Builds trust and surfaces wrong picks.

## Anti-patterns avoided

- ❌ **Don't auto-apply every suggestion silently** — list routing is high-confidence (binary: this list or that list); cleaned title and tags are subjective. Apply listId immediately, surface the rest as accept/dismiss chips.
- ❌ **Don't run LLM after insert and patch the row** — feels delayed; users see the wrong list flash before the chip appears. Inline is better.
- ❌ **Don't rely on regex routing alone** — "add to my garden list to tidy up the cut back buddleia" doesn't match `garden:` prefix. The LLM handles natural language gracefully.

## Cost

Gemini 2.0 Flash Lite via Vercel AI Gateway is ~$0.075 per 1M input tokens. Each capture is ~200-500 tokens in, ~100 tokens out. Real-world: <$0.0001 per capture. AI Gateway authenticates via `VERCEL_OIDC_TOKEN` on Vercel — no separate API key needed.

## When to use

- Single-user or small-team apps where LLM cost is negligible
- Multiple destination buckets (3+) where manual selection is friction
- Heterogeneous capture surfaces (especially voice) where cleanup adds value

## When NOT to use

- High-volume systems where LLM latency or cost is significant
- Adversarial inputs (LLM can be tricked into mis-routing — fine for personal use, not for shared orgs)
- Cases where the buckets change frequently — LLM doesn't know about lists created in the last 5 seconds without a re-fetch

## See also

- [mytodo project page](../projects/mytodo.md) — concrete implementation
- AI Gateway: https://vercel.com/docs/ai-gateway
