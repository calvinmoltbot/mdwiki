---
title: STFC Companion
type: project
repo: calvinmoltbot/stfc
created: 2026-06-12
updated: 2026-06-12
tags: [nextjs, game-tools, recommender]
---

# STFC Companion

Roster-aware crew recommender for Star Trek Fleet Command. Calvin owns ~258 of 287 officers and always struggled to pick crews; this app recommends captain+bridge+below-decks per situation (hostile grind, PvP, armadas, mining…) and per specific ship.

## Stack & layout

- Next.js (App Router, TS, Tailwind), webpack dev (`npm run dev` binds 0.0.0.0; **next.config.ts needs `allowedDevOrigins: ["100.90.11.37"]`** or remote pages render without hydration).
- All data committed under `src/data/` — builds never touch the network. No backend; roster lives in localStorage (`stfc.roster.v1`, keyed by stfc.space numeric officer ids).
- Pages: `/officers` (roster browser, bulk own, rank/level, JSON import/export), `/recommend` (use-case chips + ship picker → ranked crews rendered as an in-game-style bridge view with portraits).

## Data pipeline (the important knowledge)

- **Primary source: `data.stfc.space` — Ripper granted permission 2026-06-11.** Don't redistribute the data files; ingest output is committed so we fetch rarely. `npm run ingest` rebuilds everything (officers, ships incl. per-ship ability text from `ship_buffs` translations, ship affinity map, id migration).
- **Synergy formula** (verified): `CM_eff = CM_base × (1 + Σ captain.synergy[bridge officer class])` for same-group bridge officers. Coefficients live in the captain ability's rank slots (slot 0 = base, 1 = same-class, 2 = off-class; in `chance` fields when `ranked_is_value_chance == 1`). ⚠️ 17 absolute-unit CMs normalized slot÷base (issue #19 = verify in-game).
- **Structured mechanics** (trigger/target/modifier/conditions) merged from `pggpgg/kobayashi-stfc` (MIT), which imports Quasel's "STFC Cheat Sheet" — live at **https://stfc.cc**, currently M90; ours is M86 (issue #17).
- **Ability tags** (`src/data/abilityTags.json`): LLM-curated controlled vocabulary (12 use cases / 30 effects / 28 conditions), 287/287 coverage, validated by `scripts/merge-tags.mjs`.
- **Art**: `scripts/fetch-art.mjs` vendors portraits — repo art (2022) + **backfill from `https://assets.stfc.space/thumbs/{kind}/i/{art_id}.png`** (current for everything). Manifest-driven fallback (img onError races hydration — don't rely on it).
- **Roster capture**: no player API exists anywhere (Spock's Club sync = DLL hook in official PC client). Calvin's roster was imported from game screenshots (parallel extraction agents → name resolution → rank from per-officer level caps). Best future import: StewieDoo Officer Tool export blob (issue #16).

## Engine (`src/lib/recommend.ts`)

Score = CM relevance × synergy multiplier + OAs at owned rank, with per-use-case effect weights (`useCases.ts`), condition-match boost, ship-class conditionals, **ship-specialist handling** (abilities naming the selected ship: ×6 + relevance floor on that ship, ~zeroed elsewhere — Borg Cube → Naga Delvos/Jirali/Phlox), ship-ability keyword synergy, magnitude damper capped at 3 (absolute-unit Apex values would otherwise drown percentages). Validated against known meta crews (22 tests).

## Gotchas

- Two officers share the name "Harcourt Fenton Mudd" (different ids).
- Officer level caps per rank (5/10/15/20/30) let you derive rank from level.
- The shard-bar denominators in roster screenshots don't match the data's `shards_required` (live game rebalanced costs ~0.53×).
- gh CLI / repo conventions per Calvin's global CLAUDE.md.

## State (2026-06-12)

All original backlog (#1–#6) shipped. Open: #16–#21 (Officer Tool import, M90 mechanics refresh, exact combat formulas, synergy verification, weight tuning, automated refresh). Research briefs in `markviewer/stfc/`.
