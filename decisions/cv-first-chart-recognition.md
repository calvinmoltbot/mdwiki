---
title: "ADR: CV-First Chart Recognition (LLM for the Legend Only)"
tags: [decision, computer-vision, llm, knit-companion]
created: 2026-06-21
updated: 2026-06-21
status: active
sources:
  - markviewer/knit-companion/2026-06-21-cv-chart-recognition.md
related:
  - ../projects/knit-companion.md
  - openrouter-over-openai.md
---

# Decision: CV-First Chart Recognition, LLM for the Legend Only

## Context

knit-companion (issue #5) needs to read a knitting-pattern PDF, find the charts,
and turn them into usable interactive grids — without the user manually cropping
each one (real charts are too big to capture in one viewport). The first attempt
(slice 3) was **LLM-only**: crop a region, send it to a vision model via
OpenRouter, get back a transcribed grid.

## Problem

Vision LLMs **cannot reliably transcribe dense regular grids.** Tested directly
against the real VKF08/Snowbird Fair Isle pattern with the production
prompt/schema:

| Model | columns | rows | cells |
|---|---|---|---|
| Gemini 2.5 Flash (prod default) | 25 ❌ | 10 | all blank ❌ |
| Claude Sonnet 4.6 | 34 | 6 ❌ | partial, self-conf 0.45 |
| Gemini 2.5 Pro | 30 ❌ | 6 ❌ | garbage (purls everywhere) |

The failure is structural — column drift and row truncation when counting a dense
lattice — so **swapping models does not fix it**. It also never addressed the
original complaint: the charts are too big to crop in one viewport.

## Decision

Pivot the engine to **computer vision for the grid, LLM only for the legend.**

1. **Render** every PDF page to a pixel buffer (~2400px long edge).
2. **Detect chart regions** via XY-cut segmentation — keep only blocks that fit a
   regular square-celled lattice with a plausible cell count and a small palette;
   reject body text, photos, and legend swatches.
3. **Extract the grid deterministically** — gridline pitch = **median spacing
   between gridline peaks** (the fundamental; autocorrelation-argmax locks onto a
   harmonic, e.g. 95px = 4×24px), build a lattice, classify each cell's fill
   against a discovered palette.
4. **LLM reads only the legend** — a `mode:"legend"` call returns the yarn key
   (names + swatch hex) and stitch-symbol key. Yarn names are matched onto the
   CV palette by RGB; stitches seed the glossary. The CV grid stays
   authoritative — the model never transcribes cells.

## Why

- Knitting charts are **clean, computer-rendered grids** — crisp black gridlines,
  a tiny fixed palette — which is the *ideal* case for classical CV and the
  *worst* case for a vision LLM.
- CV grid extraction is **deterministic, exact, offline, and free.** Colour
  classification is essentially perfect (clean cluster separation).
- The LLM is kept for what it is genuinely good at — reading the **text+swatch
  legend** — where it succeeds for ~$0.0013/page.
- "Too big to capture in one viewport" **dissolves** — CV reads the whole rendered
  grid at once, no crop needed.

## Result

Validated end-to-end in Node against the real Snowbird PDF: large L-shaped charts
~75 cols (true 76), the page-4 pair 38×10 (verified geometrically correct at scan
resolution), clean teal/white/beige palette, photo/text pages correctly produce
zero charts. Legend pass returns `#54 teal (A)` / `#36 linen grey (B)` + the five
stitches. Shipped on PR #23.

**Offline-first is preserved:** with no recognition passphrase the scan is pure CV
with zero network; the legend pass is strictly additive.

## Implications / rules of thumb

- For **structured, machine-rendered visual data** (grids, tables, schematics),
  reach for classical CV first — an LLM is the wrong tool and no model swap rescues
  it.
- Use the LLM for the **unstructured, linguistic** part of the same artefact (the
  legend, free-text notes), where it is strong and cheap.
