---
title: STFC Companion
type: project
repo: calvinmoltbot/stfc
url: https://stfc.warmwetcircles.com
created: 2026-06-12
updated: 2026-07-04
tags: [nextjs, game-tools, recommender]
---

# STFC Companion

Roster-aware crew recommender for Star Trek Fleet Command. Calvin owns ~258 of 287 officers and always struggled to pick crews; this app recommends captain+bridge+below-decks per situation (hostile grind, PvP, armadas, mining…) and per specific ship.

## Deploy

- **Live: https://stfc.warmwetcircles.com** — Vercel project `stfc` under team `calvin-orrs-projects`, connected to the `calvinmoltbot/stfc` GitHub repo. Merges to `main` auto-deploy (~1 min). Wired up 2026-07-04.
- Static/localStorage app — no backend, no env vars, no secrets. `next build` on Next 16 (Vercel default bundler is fine in prod; the `--webpack` flag in `npm run dev` is only to dodge the Turbopack cross-origin dev-server hang over Tailscale, not a prod concern).
- Data reaches prod only via the committed `src/data/` bundle, so the bimonthly Data Refresh PR → merge → deploy is the update path.

## Stack & layout

- Next.js (App Router, TS, Tailwind), webpack dev (`npm run dev` binds 0.0.0.0; **next.config.ts needs `allowedDevOrigins: ["100.90.11.37"]`** or remote pages render without hydration).
- All data committed under `src/data/` — builds never touch the network. No backend; roster lives in localStorage (`stfc.roster.v1`, keyed by stfc.space numeric officer ids).
- Pages: `/officers` (roster browser, bulk own, rank/level, JSON import/export), `/recommend` (use-case chips + ship picker → ranked crews rendered as an in-game-style bridge view with portraits).

## Data pipeline (the important knowledge)

- **Primary source: `data.stfc.space` — Ripper granted permission 2026-06-11.** Don't redistribute the data files; ingest output is committed so we fetch rarely. `npm run ingest` rebuilds everything (officers, ships incl. per-ship ability text from `ship_buffs` translations, ship affinity map, id migration).
- **Synergy formula** (verified): `CM_eff = CM_base × (1 + Σ captain.synergy[bridge officer class])` for same-group bridge officers. Coefficients live in the captain ability's rank slots (slot 0 = base, 1 = same-class, 2 = off-class; in `chance` fields when `ranked_is_value_chance == 1`). ⚠️ 17 absolute-unit CMs normalized slot÷base (issue #19 = verify in-game).
- **Structured mechanics** (trigger/target/modifier/conditions) ingested live from Quasel's "STFC Cheat Sheet" (the Google Sheet behind **https://stfc.cc**) — the `RawOfficers` tab carries stfc.space officer ids, fetched via the public gviz CSV endpoint at ingest time; sheet version scraped into `meta.json` (`mechanicsVersion`, currently M90 1.7RC). `pggpgg/kobayashi-stfc` (MIT, frozen at M86) remains the fallback layer + legacy-id source. Refresh = `npm run ingest` (fails loudly on format drift). 285/287 covered (Chancellor Ake & Deidamia uncovered upstream).
- **Ability tags** (`src/data/abilityTags.json`): LLM-curated controlled vocabulary (12 use cases / 30 effects / 28 conditions), 287/287 coverage, validated by `scripts/merge-tags.mjs`.
- **Art**: `scripts/fetch-art.mjs` vendors portraits — repo art (2022) + **backfill from `https://assets.stfc.space/thumbs/{kind}/i/{art_id}.png`** (current for everything). Manifest-driven fallback (img onError races hydration — don't rely on it).
- **Roster capture**: no player API exists anywhere (Spock's Club sync = DLL hook in official PC client). Calvin's roster was imported from game screenshots (parallel extraction agents → name resolution → rank from per-officer level caps). The /officers Import panel now also accepts the StewieDoo Officer Tool "Officer Export Data" TSV blob (`src/lib/officerToolImport.ts`: 64-entry alias table, Mudd disambiguation by id, rank derived from level caps; merges, never un-owns). Caveat: the tool's sheet is v1.8.M87, so the 9 newest officers can't arrive via that blob.

## Engine (`src/lib/recommend.ts`)

Score = CM relevance × synergy multiplier + OAs at owned rank, with per-use-case effect weights (`useCases.ts`), condition-match boost, ship-class conditionals, **ship-specialist handling** (abilities naming the selected ship: ×6 + relevance floor on that ship, ~zeroed elsewhere — Borg Cube → Naga Delvos/Jirali/Phlox), ship-ability keyword synergy, magnitude damper capped at 3 (absolute-unit Apex values would otherwise drown percentages). Validated against known meta crews (22 tests). **Applicability gate (issue #36):** `isApplicableToUseCase` (in `abilityApplicability.ts`) hard-zeros an ability when Quasel's matrix rates it ➖ in every context a use case maps to — wired into `abilityScore`, limited to `pvp`/`armada` (the only use cases that gate cleanly; mining/hostile/station contexts conflate modes), ship-specialists exempt.

**Combat math (issue #18, 2026-06-12):** `src/lib/combatMath.ts` is a clean-room kernel of the community formulas (mitigation S-curve `1/(1+4^(1.1−d/p))` with class coefficients 0.55/0.2/0.2, DPR with crit + load/reload amortization, shield/hull split → EHP, stacking `A·(1+B)+C`, iso/apex math ready but unwired, status constants). Inputs: `src/data/shipStats.json` (per-tier profiles, all 114 ships) + `hostiles.json` (445 median representatives of 5,381, per level×class×rarity) via `src/lib/combatData.ts`. `src/lib/marginalValue.ts` derives 16 combat effect weights analytically at module load (ΔDPR × Δsurvival at median ability magnitudes — the S-curve is convex, so uniform small deltas mislead); remaining weights stay heuristic, provenance in `WEIGHT_PROVENANCE` (useCases.ts). Known gaps (flagged in PR #27): no iso/apex stats in data, `healthByLevel` semantics unverified, no fight-length model for regen, condition uptime ignored.

## Gotchas

- Two officers share the name "Harcourt Fenton Mudd" (different ids).
- Officer level caps per rank (5/10/15/20/30) let you derive rank from level.
- The shard-bar denominators in roster screenshots don't match the data's `shards_required` (live game rebalanced costs ~0.53×).
- gh CLI / repo conventions per Calvin's global CLAUDE.md.

## State (2026-06-12)

All original backlog (#1–#6) shipped. Shipped 2026-06-12: #16 (Officer Tool import, PR #23), #17 (M90 mechanics refresh, PR #22), #21 (Data Refresh workflow_dispatch button + bimonthly cron, PR #25, verified end-to-end), #18 (combat math: stats ingest PR #26 + kernel PR #27). Also shipped 2026-06-12 (PRs #32–#35): #24 applicability matrix ingest + cross-validation script (`scripts/check-applicability.mjs` — 87.9% agreement, 413 disagreements incl. genuine tag bugs: TMP Sulu, Severus, Cadet Uhura; scoring integration still open), #28 iso/apex layer (4 new vocabulary tags, 67 retags, mid-curve operating points — no ship/hostile iso/apex stats exist in the source, only player research → `isoApex.json`), #31 condition uptime model (`conditionUptime.ts`: 28 conditions classified, kernel fight profiles ~1.9/6.2/0.8 rounds for hostile/armada/PvP) + analytic regen, #30 crit operating point (`critResearch.json` mined like iso/apex; weights at research-stacked profiles — Mirror Data restored to #1 grind). Engine tests now 159. Shipped 2026-06-13: #36 (act on applicability cross-validation, branch `feature/36-applicability-crossval`): all 3 tiers — fixed 3 known tag bugs + 2 more, two systematic mapping corrections in ingest (`Utility⊇{mining,cargo}`, `PvPStation⊇pvp`), full agent-assisted per-officer review (60 useCases edits / 53 officers), and the tier-3 engine gate (`isApplicableToUseCase`, pvp/armada only). Tag↔matrix agreement 87.9%→**95.5%** (413→154 residual, all defensible: armada solo/group granularity, loot-reason cells, unmapped modes). Tests 159→164. Triage report: `markviewer/stfc/2026-06-13-applicability-crossval-triage.md`. Open: #19 (synergy verification), #20 (weight tuning), #29 (healthByLevel semantics) — all three need Calvin in-game. Combat-math research brief: `markviewer/stfc/2026-06-12-stfc-toolbox-combat-formulas.md`. Research briefs in `markviewer/stfc/`.
