---
title: golf — Holywood Golf tournament manager
created: 2026-05-18
updated: 2026-05-24
status: active
tags: [golf, firebase, react, vite, vercel]
related:
  - ../systems/firebase-functions-iam-gotcha.md
  - ../systems/gcloud-headless-auth.md
---

# golf

Tournament management app for **Holywood Golf Club's Red Arrows weekly sweep**. Built so Calvin's contact at the club (non-technical) can enter scores via a magic link, while Calvin and other admins manage draws, players, and the season from the web app.

For live feature state, use `gh issue list --repo calvinmoltbot/golf` — not this page.

## URL & infra

- Live: **https://golf.warmwetcircles.com** (and any other custom domains in Vercel)
- Repo: `calvinmoltbot/golf`
- Vercel project: under `calvin-orrs-projects`
- Backend: **Firebase project `holywoodgolf`** (Firestore + Cloud Functions + Auth)
- Functions region: `us-central1`
- Service-account JSON for admin scripts: `./Ignore/holywoodgolf-firebase-adminsdk-*.json` (gitignored)

## Tech

- Vite + React + TypeScript (no Next.js)
- Firestore for storage, Firebase Auth (Google OAuth) for login
- 1st-gen Cloud Functions (callables + a few HTTP), Node 20 — **deadline 2026-10-30 to upgrade** (issue #32)
- Tailwind, lucide-react icons
- pnpm workspaces (root + `functions/`)
- Tests: Node.js test runner, in `tests/`
- No `react-router` — routes matched by pathname regex in `src/App.tsx`

## Design language — "Fairway glass"

As of 2026-05-24 the **whole app** is on one design language (rolled out across #88–#101). Tokens live in `src/index.css` (CSS vars) + `tailwind.config.js`:
- Colours: `board-green` (#1e4a31, primary), `paint-red` (#b03326, **money/winners/destructive only**), `paint-yellow` (#e7b13a, highlight/caution — never as text colour); neutrals `admin-bg`/`admin-surface`/`admin-border`/`admin-text`/`admin-muted`.
- Hover/ink tokens (added #95): `board-green-hover`, `paint-red-hover`, `paint-yellow-ink` (#8a6a1f — the readable yellow for text on yellow tints), `admin-border-hover`. **Use these, don't hard-code hover hexes.**
- Patterns: gradient accent strip `bg-gradient-to-r from-board-green via-paint-yellow to-paint-red`; frosted glass `bg-white/70 backdrop-blur-xl`; `rounded-full` pill controls; inner content cards `rounded-md border border-admin-border bg-admin-surface`, with `rounded-2xl` reserved for one hero card per screen; `font-display` titles / `font-ui` body.
- Gold-standard reference components: `StandingsTab`, `AdminTopBar`, `DesktopHomePreview`.

## Data model (Red Arrows sweep)

- **`SweepTournament`** = a season (e.g. "Red Arrows 2026")
- **`MonthlyDraw`** = one month's teams + four match dates (Saturdays)
  - Path: `users/{userId}/sweepTournaments/{tournamentId}/draws/{drawId}`
- **`Match`** = one Saturday's scoring round, has `results: RedArrowsResult[]`
  - Path: `…/draws/{drawId}/matches/{matchId}`
  - `matchNumber` is 1–4 within a draw; `date` is YYYY-MM-DD as ISO string
- **`Player`** under `users/{userId}/players/{playerId}`
- Tokens at top-level: `scorerTokens/`, `publicShareTokens/`

Player IDs are doubly disconnected from the global Red Arrows player DB — see ROADMAP.md.

**Cadence is monthly.** One `MonthlyDraw` = the same five teams for that month's ~4 Saturdays; each Saturday is its own `Match` for scoring. The club briefly drew *weekly* early in the 2026 season (so a one-off weekly draw like 25 Apr exists as its own draw doc `draw-2026-04-25`), then settled on monthly. **The monthly-draw generator can't skip a Saturday** — it always creates 4 consecutive weekly matches from `firstMatchDate`; any skipped/extra week needs a manual data-fix script.

## Draw engine (`src/utils/redArrowsDrawEngine.ts`)

Builds the five teams for a draw. Two variety rules, both **soft preferences** (always produces a draw):
1. **President** (`SweepTournament.currentCaptainId`) leads Team 1 and plays with everyone once over the season — rotates onto people he hasn't been paired with yet.
2. **All five teams** minimise repeat pairings: the engine tracks `pairCounts` (how often each pair has shared a team), generates ~250 candidate draws, keeps the one with most president-new-partners then lowest total repeat cost, then runs local-search swaps between non-captain teams.

**The history must be fed at the call site.** The engine derives its rotation/pairing state from `season.weeks`. `MonthlyDrawCreator` builds those from prior draws via `drawsToHistoryWeeks(existingDraws, firstSat)` (in `drawPlayerAdapter.ts`) — draws strictly *before* the month being drawn. (Originally the call site passed `weeks: []`, so rotation silently never worked in prod — PR #83.) Variety-across-all-teams shipped in #84; remaining polish (on-screen variety score) tracked in **#84**. Analysis tools: `scripts/drawAudit.ts`, `scripts/pairingFromPdfs.ts`.

## Three token-gated public routes

All unauth — see [firebase-functions-iam-gotcha](../systems/firebase-functions-iam-gotcha.md) for the deploy quirk that bites these.

| Route | Token store | Cloud Function | Purpose |
|---|---|---|---|
| `/score/:token` | `scorerTokens/` | `getScorerRoundContext` + `submitScores` | Admin enters scores from phone |
| `/live/:token` | `publicShareTokens/` | `getPublicShareContext` | Public mobile leaderboard (QR-shareable) |

## Deploying

- **Frontend**: push to `main`, Vercel auto-deploys.
- **Functions**: `pnpm --filter functions run deploy` — wraps `firebase deploy` and re-grants `allUsers` invoker on the three unauth callables (`submitScores`, `getScorerRoundContext`, `getPublicShareContext`). Don't use raw `firebase deploy` without re-granting — it strips public access every time.
- **Rules**: `firebase deploy --only firestore:rules --project holywoodgolf`
- **gcloud auth**: account `calvinmoltbot@gmail.com` does NOT have rights on the `holywoodgolf` project. The actual owner is `calvin.orr@gmail.com`. For IAM operations, use the Firebase Console (the SA in `Ignore/` lacks Cloud Functions admin permissions).

## Admin scripts (one-off recovery)

In `scripts/` — read-only or dry-run by default; writes need `--commit`/`--apply`. **SA prod *reads* are also blocked by Claude's auto-mode classifier — Calvin runs them via the `!` prefix.** All take `--service-account Ignore/holywoodgolf-firebase-adminsdk-*.json`.

- `scripts/fixMatchDate.ts` — list matches by date (collectionGroup scan) or rewrite a single match's `date` field in place.
- `scripts/moveMatchResults.ts` — atomically move `results` array between two Match docs. Marks destination `completed`, resets source to `scheduled`. Refuses to clobber.
- `scripts/drawAudit.ts` — read-only: dump season captain, captain memberships per sweep, president-partner history (flags repeats), each draw's teams + matches, and a **who-played-whom / repeated-pairings** table.
- `scripts/pairingFromPdfs.ts` — local (no Firestore): pairing analysis from hardcoded draw data, used as ground truth from the printed sheets.
- `scripts/fixMarchBlockMatches.ts`, `scripts/backfill25April.ts` — the one-off 2026 corrections (drop phantom 4 Apr, add 25 Apr; record 25 Apr as its own draw with real teams). Records of past surgery — not for reuse.

Issue **#30** tracks proper admin-UI replacements for these.

## Key files

- `src/types/index.ts` — domain models (lines 700–760 for Match/MonthlyDraw)
- `src/utils/redArrowsStats.ts` — leaderboard aggregation (`computePlayerStandings`)
- `src/services/firebase.service.ts` — Firestore CRUD
- `src/services/scorerToken.service.ts`, `src/services/publicShareToken.service.ts`
- `src/components/home/DesktopHomePreview.tsx` — **the live Home tab** (named export `StandingsHeroHome`, standings-as-hero + match panel/Quick Actions; default export is the `/home-preview` route). NB the old `HomeDashboard.tsx` was deleted as dead code (#97).
- `src/components/ExtrasMenu.tsx` — the **More tab** (tile grid + per-section render switch; add new admin tools here as a tile + a `renderSubContent` case)
- `src/components/admin/RotationToolsLoader.tsx` — self-loading wrapper feeding `SeasonSimulator`/`AdminOverview` the active sweep + `drawsToHistoryWeeks(draws)` history
- `src/components/scorer/LiveSharePage.tsx` — the `/live/:token` view (mobile-first template to mirror for #33)
- `src/components/scorer/PublicShareAdmin.tsx` — admin UI for minting share tokens (with QR)
- `functions/src/index.ts` — function exports
- `functions/src/scorerToken.ts`, `functions/src/publicShare.ts` — callable impls
- `functions/scripts/deploy-with-iam.sh` — the wrapper, with `PUBLIC_FUNCTIONS` array

## Gotchas

- **⚠ Match doc IDs repeat per draw — NOT unique across the season.** Matches get deterministic IDs `match-1`..`match-4` *per draw* (`createMonthlyDraw`), so every draw has its own `match-1`. Any code that flattens matches across draws and keys on bare `match.id` / `matchId` collapses draws onto each other. **Always key on the composite `${tournamentId}/${drawId}/${matchId}`** (or at least `drawId/matchId`). This has bitten three times: Home prev/next nav (#72, later months unreachable), scorer-link candidate selection + scoring-timeline React keys (#79), and `listTokensForMatch` returning other draws' tokens (#82). When touching anything that lists/selects/aggregates matches season-wide, assume this trap until proven otherwise.
- **gcloud auth on the headless Mini** for the functions deploy's IAM step — see [gcloud-headless-auth](../systems/gcloud-headless-auth.md). `calvin.orr@gmail.com` (project owner) must be the active gcloud account; `gcloud auth login --no-launch-browser` in a *real* terminal (not Claude's `!`); prod deploy needs explicit user OK.
- **Reverting a match to "no scores"**: the AdminMatchEditor per-row `×` button *marks a player absent* — it does not cleanly delete the result. To truly reset, write `{results:[], absentPlayerIds:[], status:'scheduled'}` to the match doc via an admin-SDK script (SA prod writes are blocked by Claude's auto-mode classifier, so run via `!`).
- **Collection-group queries** on Match's `date` field require a single-field collectionGroup index that isn't provisioned by default — admin scripts filter client-side instead.
- **`where + orderBy`** on top-level collections (e.g. `publicShareTokens`) needs a composite index. Default to client-side sorting unless data volume warrants the index.
- The Firebase admin SDK service account does NOT have `cloudfunctions.functions.get` — use the Firebase Console for IAM, not `gcloud` as that SA.
- Node 20 runtime deprecation hard deadline: **2026-10-30**.
- **History-derived engine state must be fed at the call site.** The draw engine computes rotation/variety from `season.weeks` — if the screen passes `weeks: []`, it silently behaves as if there's no history (this shipped broken for months; PR #83). When debugging "the draw ignored history," check the call site (`MonthlyDrawCreator` / `drawsToHistoryWeeks`), not just the engine.
- **Hard-refresh before drawing after any out-of-app data change.** `MonthlyDrawCreator` draws against the draw list held in the browser's React state. After running an admin script (e.g. backfilling a draw), an un-refreshed tab will draw against stale history and miss it. Cmd+Shift+R first.
- **PDF "president" must key on `currentCaptainId`, not a `captain` membership role.** Players keep a `captain` `sweepMembership` from past seasons, so checking `role === 'captain'` flags ex-presidents too (two red dots). `matchPdfExport.ts` now scopes to `tournament.currentCaptainId`.
- **Player nicknames vs real names** (system vs club scorecard): "Al Jones" = **Alister Jones** (current president); "Potsy Nevin" = **Garth Nevin** (2025 ex-president, hence a leftover captain membership); "Dave Brown" = Davy Brown; "Tony Denvir" = Anthony Denvir.
- **Vercel auto-deploy can lag** a few minutes on a `main` push (it's not broken). To force a prod deploy: `vercel --prod --yes` (builds with prod env vars, re-aliases the domain).
- **Before deleting "dead" UI, grep for the service calls it uniquely owns.** The May-19 IA pivot (#41/#47) unrouted several components but left the files behind, so they *looked* like deletable orphans. Twice in one session (2026-05-24) an "orphan" turned out to be the **sole entry point for a needed capability**: the president-rotation tools (`SeasonSimulator`/`AdminOverview`) and — critically — `SweepTournamentManagement`, the *only* UI that calls `createSweepTournament` (delete it and there's no way to start a new season; Sweep Admin's empty-state even pointed back at the dead "Tournaments tab"). All three were **reconnected**, not deleted (#100/#101), as admin-gated tiles in More. Lesson: `grep -rn createX` the unique service call before deleting; an unmounted component ≠ an unneeded one. (Genuinely-dead cruft removed separately in #97/#98.)
- **`vite dev` is unusable over Tailscale** (~1900 separate module requests, each a tailnet round-trip → page never loads). Use `pnpm build` + `vite preview --host 0.0.0.0`, or just deploy. Firebase OAuth only authorises `localhost` by default (not the Tailscale IP or `*.vercel.app` previews) — test real auth on `golf.warmwetcircles.com` or via an SSH `-L` tunnel to localhost. The in-app dev-bypass logs in as `dev-user` (empty data) — useless for verifying real draws.
