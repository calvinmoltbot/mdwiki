---
title: golf — Holywood Golf tournament manager
created: 2026-05-18
updated: 2026-05-21
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

In `scripts/` — both default to dry-run, require `--commit`:

- `scripts/fixMatchDate.ts` — list matches by date (collectionGroup scan) or rewrite a single match's `date` field in place.
- `scripts/moveMatchResults.ts` — atomically move `results` array between two Match docs. Marks destination `completed`, resets source to `scheduled`. Refuses to clobber.

Issue **#30** tracks proper admin-UI replacements for these.

## Key files

- `src/types/index.ts` — domain models (lines 700–760 for Match/MonthlyDraw)
- `src/utils/redArrowsStats.ts` — leaderboard aggregation (`computePlayerStandings`)
- `src/services/firebase.service.ts` — Firestore CRUD
- `src/services/scorerToken.service.ts`, `src/services/publicShareToken.service.ts`
- `src/components/RedArrowsMainDashboard.tsx` — admin landing page
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
