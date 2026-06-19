---
title: Knit Companion — pattern-following PWA
created: 2026-06-19
updated: 2026-06-19
status: active
tags: [nextjs, react, typescript, pwa, indexeddb, pdfjs, knitting, data-model]
related:
  - ../patterns/nextjs-use-server-export-rule.md
---

# Knit Companion

A web app for **following knitting patterns** — load a pattern PDF, track your place,
and count rows. Local-first PWA: everything lives in the browser, no backend.

Local path: `~/Dev/Projects/knit-companion`. Repo: `calvinmoltbot/knit-companion`.
For live feature state use `gh issue list --repo calvinmoltbot/knit-companion` — not this page.

## Stack

**Next.js 16** (App Router, client-heavy) · **React 19** · **TypeScript** · **Tailwind 4**.
PDF rendering via **pdf.js** (worker copied to `public/pdf.worker.min.mjs` by a postinstall
script). Persistence via **IndexedDB** (the `idb` library) — projects and raw PDF bytes are
stored in separate object stores, keyed by project id. No server, no sync.

⚠️ This repo's `AGENTS.md` warns that its **Next.js has breaking changes vs training data** —
read `node_modules/next/dist/docs/` before writing Next code here, don't assume.

## Two layers: PDF-only MVP, and a structured pattern model

The original MVP models a pattern as a **dumb PDF**: page number, a draggable row-marker
band (per page), and flat manual counters (`Counter` with optional `target`/`wrapAt`). That
still works and is the fallback.

Layered on top (optional, additive — merged Jun 2026, PR #10) is a **structured
`Pattern`** in `lib/types.ts`: the app can offer size-aware instructions, section
navigation, charts, and a glossary instead of just a PDF. A `Project` gains optional
`pattern` / `selectedSize` / `position`; absent = PDF-only behaviour. Helpers
(`resolveSize`, `formatSized`, `renderLine`, `validatePatternSizing`) live in `lib/pattern.ts`.

The **viewer** that renders this (issue #8, PR #11) is `components/PatternViewer.tsx` —
a pure/controlled component (caller owns `selectedSize`). It does the size picker,
size-resolved meta/measurements, `appliesToSizes` variant filtering (the "only show me
my size" win), token highlighting (selected size bold, others dimmed), per-size `repeat`
summaries, "at the same time" badges, chart/stitch-pattern reference chips, and
tap-to-define glossary. Reached today via `app/pattern/[id]/page.tsx`, a sample-only
route over `SAMPLE_PATTERNS` that owns `selectedSize` in localStorage (samples aren't DB
projects); the component is built to later drop into the real project page driven by
`Project.selectedSize`. Charts render as *references* only — grids are #4.

## The core design insight: sizing is not one thing

The model was designed against **three deliberately different reference patterns**
(`lib/fixtures/`, registered in `SAMPLE_PATTERNS`), because sizing works three distinct ways
and a real app must handle all of them:

- **Per-stitch** — *Summer Sorrel* (14-size top-down tee): almost every number varies by
  size. Modelled by `SizedValue<T> = T | (T|null)[]` (the spine of the schema); `null` = the
  printed `-` ("not worked for this size"). Also has size-grouped line variants
  (`Line.appliesToSizes`) and a parallel colour fade (`Section.parallel` / `runsAlongside`).
- **Length-based** — *Veronya Warmer* (flat headband, 3 sizes): identical stitch
  instructions; size only changes a **repeat length target**. Modelled by `Repeat`
  (row-range, count- *or* measurement-terminated, with `endOnRow`).
- **Single-size** — *Snowbird* (Fair Isle mittens): the machinery degrades to N=1. Its
  charts are **dual-axis** (`ChartCell` carries both a stitch `symbol` and a colourwork
  `color`), grid-only (no written rounds), and 76×50 — too big to hand-enter.

Other concepts the patterns forced into the schema: `Yarn` (A/B colourwork), `StitchPattern`
(named reusable motifs like Corrugated Rib), `Chart` with *either* written rounds and/or a
grid, and `PatternMeta` (construction, skill level, notions, gauge, measurements).

## Gotchas / hard rules

- **Sample PDFs are copyrighted** ("personal use only — no distribution"). They are
  **gitignored** (`/public/samples/*.pdf`), kept local as dev fixtures only, and must never
  be bundled or deployed (Next would serve them at `/samples/*.pdf`). Ship freely-licensed
  samples before any public deploy (issue #9). The structured `.ts` fixtures are full
  transcriptions — treat as a dev/test corpus, not shippable content.
- **Fixture numbers are hand-transcribed** from the PDFs — good as a regression corpus,
  verify against source before relying on any number for actual knitting.
- **Big chart grids are left un-transcribed** on purpose (Snowbird, Summer Sorrel) — the
  realistic path is chart recognition from a PDF region (issue #5), not hand-entry.
- Validate fixtures after edits: `npx tsx -e` calling `validatePatternSizing` over
  `SAMPLE_PATTERNS` (checks every per-size array has `sizes.length` entries).

## Backlog shape

The structured model reframes most open issues as facets of one thing: #4/#5 (charts —
grid + recognition), #6 ("at the same time" → parallel sections), #7 (magic markers →
glossary refs / `LineToken`), #1 (PWA/offline). Dev-only target for all of these is the
`SAMPLE_PATTERNS` registry.

**#8 (size picker + structured viewer) landed** in PR #11 (Jun 2026). It surfaced a
follow-up, **#12**: most per-stitch fixture lines are still plain `text` (not tokenized),
so the full printed run shows inline — the viewer's "only my size" win currently comes
mainly from `appliesToSizes` filtering + per-size meta/repeat resolution. #12 is to
tokenize those lines (or parse runs) so non-selected sizes collapse inline.
