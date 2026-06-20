---
title: Knit Companion — pattern-following PWA
created: 2026-06-19
updated: 2026-06-20
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

## Benchmark: knitCompanion (the app we're emulating)

We're explicitly emulating **knitCompanion** (knitcompanion.com, App Store id1058142783,
Create2Thrive) — but as a **mobile-first web PWA first**, OS-app question deferred. UI study
(CSS reconstructions of its iPhone screens + pattern→roadmap mapping):
`markviewer/knit-companion/2026-06-19-knitcompanion-ui-study.html`.

kC is **chart-centric, PDF-overlay**. Its whole experience is one loop *on a chart grid*:
see chart → a bold **row band** marks the current row → **one-tap (or voice) advance** →
on-chart **magic markers** + colour-coded **edge counters** hold your place. Supporting:
stitch key w/ plain-language defs, scribble layer, mode toggle (chart↔written), Join tool,
calculators. Their paid **kCDesigns** (pick size → numbers highlighted, markers pre-set) ==
**our structured `Pattern` model** — that's our built-in edge. Voice control (Web Speech API,
~7-word grammar NEXT/BACK/…) noted as a cheap future win but **parked** for now.

## Backlog shape

The structured model reframes most open issues as facets of one thing: #4/#5 (charts —
grid + recognition), #6 ("at the same time" → parallel sections), #7 (magic markers →
glossary refs / `LineToken`), #1 (PWA/offline). Dev-only target for all of these is the
`SAMPLE_PATTERNS` registry.

**#8 (size picker + structured viewer) LANDED + MERGED** — PR #11 squash-merged to main
2026-06-20 (`4e51880`), #8 closed, branch deleted. `components/PatternViewer.tsx` +
`app/pattern/[id]/page.tsx` are now on main.

**#4 re-scoped as the KEYSTONE** (2026-06-20) — retitled "Chart / grid mode — interactive
grid + row band (keystone)". It's the kC core loop: render `Chart`/`ChartCell` (dual-axis
symbol+colour) as a real steppable grid, row band tied to `Project.position`, one-tap
advance, big current/total row counter, size-aware, tap-to-define symbols. Manual/fixture
charts first. Deferred *out* of #4: on-chart markers+edge counters → **#7** (depends on the
grid), auto-recognition → **#5**, Join tool (unscoped), voice control (parked).

**#4 (chart grid — KEYSTONE) LANDED** — PR #13 open (branch `feature/4-chart-grid`,
2026-06-20), Closes #4. `components/ChartGrid.tsx` is the steppable surface: dual-axis
cells (stitch `symbol`→glyph and/or colourwork `color`→yarn swatch), a bold row band on
the current round, reading-direction aware (`worked.fromBottom` flips display order so
round 1 is at the bottom, `rightToLeft` flips cells + stitch numbers so st 1 is on the
right), a big `current / total` counter with −/+ and tap-to-advance, size-aware repeat
chips, tap-a-stitch→glossary sheet, auto-built key. Empty grids degrade to a dimensions
placeholder (so Snowbird's deferred charts don't break). Integrated in
`app/pattern/[id]/page.tsx` via a Pattern↔Chart mode toggle + chart picker, with the
current round persisted **per chart** to localStorage (key `knit-sample-state:<id>`,
migrates the old `knit-sample-size:<id>` from #8). `PatternViewer` chart chips became
tappable launchers (new optional `onOpenChart` prop).

**New fixture: `lib/fixtures/stitch-sampler.ts`** — added because EVERY existing chart
`grid` was empty (Snowbird's deferred to #5), so chart mode had nothing to render. It's
grid-first, **traditional / public-domain** (Old Shale lace 18×4 + Fair Isle X-and-O
colourwork 8×9), **no source PDF** — so it's deploy-clean (unlike the copyrighted PDF
samples). Symbol-key contract the renderer expects: `k`/`p`/`yo`/`k2tog`/`ssk`/`sk2p`
(+`nostitch`), each defined in `glossary` with a matching `StitchDef.symbol`; unknown keys
fall back to rendering the raw string. `ChartGrid` maps colourwork yarn ids → display
swatches by yarn order (the model has no colour field).

### NEXT STEPS (post #4, 2026-06-20)
- **#7 (magic markers / edge counters) is now UNBLOCKED** — it was waiting on the grid to
  exist; markers + colour-coded edge counters live *on* `ChartGrid`. Likely next build.
- **#12** still open (independent): per-stitch fixture lines are plain `text` not tokenized,
  so the full printed run shows inline; tokenize so non-selected sizes collapse.
- **#5** (chart recognition from a PDF region) is the realistic path to fill the big empty
  grids (Snowbird, Summer Sorrel) that were left un-transcribed on purpose.
