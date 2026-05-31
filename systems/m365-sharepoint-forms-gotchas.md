---
title: M365 SharePoint + Forms build gotchas
created: 2026-05-31
updated: 2026-05-31
status: active
tags: [m365, sharepoint, microsoft-forms, power-automate, no-code, gotcha]
related:
  - ../projects/capital-programme-system.md
---

# M365 SharePoint + Forms build gotchas

Build limits of the no-code Microsoft 365 stack (Forms → SharePoint lists → Power
Automate) that shape design decisions. First hit on
[Capital Programme System](../projects/capital-programme-system.md).

## 1. Microsoft Forms has no grid / matrix question

There is **no table/grid/matrix numeric question type** in Microsoft Forms. (Likert
exists but is single-choice-per-row, not numeric entry — useless for £ amounts.)

So any conceptually-tabular input (e.g. a 3-years × 3-funding-types funding profile =
9 cells) must become **individual Number questions** — one per cell.

### Branching to avoid a wall of inputs

To stop managers facing nine number boxes (most often 0), gate inputs behind
choice questions and branch. **But Forms branching is section-level and the
"Go to" at the end of a section is STATIC** — it cannot look back at an earlier
answer to decide where to route next.

- ❌ A single 3-way gate ("Council only / External only / **Both**") breaks: it can
  route *into* the Council section, but can't conditionally decide "after Council,
  also do External (Both) vs skip to Revenue (Council-only)".
- ✅ Use a **Yes/No gate per source**, each immediately before its own section. After
  a source's section, flow just continues to the next gate. "Both" = Yes to two
  gates. Composes cleanly with section branching, no look-back needed.

**Sectioning is required for branching** — each gated block must be its own Forms
section.

### Other Forms limits worth knowing

- **Can't enforce a cross-question rule** like "at least one of Q14/Q15 = Yes".
  Handle downstream: have the Power Automate flow reject the submission (e.g.
  `Total = 0`) and email back, rather than relying on Forms validation.
- Forms can't show a computed running total — that's mockup-only UX. Totals land on
  the SharePoint list (Calculated columns) and any generated PDF.

## 2. SharePoint "Create list → From Excel" only makes 4 column types

Uploading an `.xlsx` (formatted as an Excel **Table**) via *New → List → From Excel*
infers column types from the data — but **only ever produces**: Single line of text,
Number, Date, or Yes/No.

Everything richer is a **manual pass after import**:

- **Choice** → imports as Single line of text; convert afterward (and add the choices).
- **Person or Group** → imports as text; can't be converted from text. Add the Person
  column by hand, then populate.
- **Calculated** → never imported. Add manually with the formula once the source
  number columns exist.
- **Multiple lines of text** → imports as Single line; change the column type after.
- Entirely-empty columns may be skipped on import — don't rely on import to create a
  column that has no example data.

**Practical pattern:** make the import file carry only the cleanly-inferred,
data-bearing columns (+ a couple of realistic example rows so types infer right), and
ship a companion **"Column setup" sheet** that specs every column and flags which are
*Import* vs *Convert after* vs *Add manually*. The import gives you the list + the
straightforward columns fast; the setup sheet makes the manual remainder mechanical.

## 3. Power Automate "Populate a Word template" maps by content-control title

The *Populate a Word template* action (Word Online (Business)) surfaces each
**plain-text content control** in the template by its **Title (`w:alias`)**. So:

- You don't need last year's template or its control names — **ship your own template
  and name each content control after the destination field**. The mapping then
  becomes a 1:1 lookup in the flow.
- docx-js can't emit content controls directly; build the doc skeleton with
  placeholder tokens, then inject `<w:sdt>` (alias + tag + id + `<w:text/>`) wrappers
  via an XML pass. (See `scripts/inject_content_controls.py` in the capital project.)
