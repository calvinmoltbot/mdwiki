---
title: Capital Programme System — council bid & gateway tracker
created: 2026-05-31
updated: 2026-05-31
status: active
tags: [m365, sharepoint, power-automate, power-bi, microsoft-forms, council, no-code]
related:
  - ../systems/m365-sharepoint-forms-gotchas.md
---

# Capital Programme System

A council **capital-programme bidding and gateway-review system** for FY 2027/28,
built entirely on **Microsoft 365 / Power Platform** — no custom code app. Replaces
a single ~50-column SharePoint list with a narrow current-state list plus an
append-only events list, so every bid's gateway journey is auditable and Power BI
can report both current state and gateway flow/attrition.

Local path: `~/Dev/Projects/Capital2728`. Repo: `calvinmoltbot/capital-programme-system`.
For live feature state use `gh issue list --repo calvinmoltbot/capital-programme-system` — not this page.

## Stack (M365, no Power Apps)

**Microsoft Forms** (capture) → **SharePoint lists** (storage) → **Power Automate**
(automation) → **Power BI** (reporting). No front-end code. Self-service account
creation / permission changes are out of scope.

## The core design move

Store a scheme's gateway history as **rows in an append-only events list**, not as
ever-multiplying columns on the scheme record. The scheme record stays narrow and
holds current state only. Three lists:

- **Schemes** — one row per bid; current state; classification; **9 fixed funding
  columns** (3 years × Council/External/Revenue).
- **Gateway Events** — many rows per bid; append-only audit trail; the funnel source.
- **Directory** — one row per directorate; routing lookup (Director, BusinessPartner).

## Hard rules (don't violate without instruction)

1. **No Power Apps** — front door is Microsoft Forms (tried twice, failed on build
   time + adoption).
2. **Funding profile stays as COLUMNS** (9 fixed cells), not child rows. Forms writes
   one flat row; unpivot in Power Query at report time. *Rows-not-columns applies
   ONLY to gateway history.*
3. **Schemes are never deleted** — a declined/deferred/withdrawn scheme gets a
   terminal *event row*; deleting breaks funnel/attrition reporting.
4. **SchemeType is finance-set**, never on the manager form.
5. **No AI judgement of bids** — AI may draft decision-email wording (human-reviewed),
   but must not score, rank, or evaluate (audit defensibility).

## Build artifacts (in repo, generated)

- `schemas/SharePoint-Lists.xlsx` — canonical column schema (the source of truth).
- `schemas/import/*.xlsx` — "Create list → From Excel" templates (Import table +
  Column-setup sheet), via `scripts/build_import_files.py`.
- `templates/scheme-summary-template.docx` — Flow A confirmation PDF template with
  34 plain-text content controls **named after each Schemes field** (so Power
  Automate's "Populate a Word template" maps 1:1). Built by
  `scripts/build_word_template.js` + `scripts/inject_content_controls.py`.
- `docs/flow-a-build-spec.md` / `docs/flow-a-emails.md` — Power Automate Flow A
  click-by-click + email bodies.
- `docs/form-build-guide.md` — Microsoft Form question list, sectioning, and
  **branching** rules.

## Phasing

- **Phase 1 (mid-June, critical):** Schemes list, Directory list, Form, Flow A
  (capture/route/PDF/confirm), optional soft profiling nudge.
- **Phase 2 (Jul/Aug):** Gateway Events list, Flow B (decision comms), Gateway-2
  profiling flag.
- **Phase 3:** Power BI state + funnel reports, Power Query unpivot of the 9 funding
  columns, optional AI email drafting.

## Gotchas

The M365 build limits that shaped the design (no Forms grid, branch-per-source;
SharePoint "From Excel" column-type limits) are written up in
[M365 SharePoint + Forms gotchas](../systems/m365-sharepoint-forms-gotchas.md).

## Current state (2026-05-31)

Phase 1 fully **specced and import-ready on `main`** (PR #13 merged) — Form (branched),
list import templates, Flow A spec, and the Word template all done. Remaining Phase 1
steps need the live council M365 tenant: import the lists, build the flow, publish the
form. A tenant dry-run was the next action.
