# OpenRouter Tracker

**Repo:** calvinmoltbot/openrouter-tracker
**Live:** openrouter.warmwetcircles.com
**Stack:** Next.js 16 (App Router) + React 19 + Tailwind CSS 4 + shadcn/ui + Chart.js + Drizzle ORM + Neon DB

## Architecture

Dashboard for visualizing OpenRouter API spending. Three zones: Dashboard (overview KPIs + charts), Breakdown (model/app deep-dive), Heatmap (usage patterns by model/app/key).

**Data flow:** Server fetches OpenRouter Activity API via `OPENROUTER_PROV_KEY` → client calls `/api/activity` → `processRows()` → `ProcessedData` → zones render charts. Also supports CSV upload and live webhook ingestion (`/api/webhook/openrouter`).

**Key directories:**
- `src/app/` — Routes, API proxies, layout, globals.css
- `src/components/` — Dashboard shell, zone tabs, KPI cards, chart wrappers
- `src/lib/` — Types, formatters, chart configs, theme logic
- `src/data/` — Data processor, color palette
- `src/db/` — Drizzle schema (logs table + auth)

## Deployment

Vercel auto-deploys from `main`. CI runs lint + build on push/PR. Config in `vercel.json` (`"framework": "nextjs"`). Secrets (`OPENROUTER_PROV_KEY`, Postgres URL) set in Vercel dashboard.

## Gotchas

- **Dark mode theming:** Theme class toggled on `<html>` root. Inline script in `layout.tsx` reads localStorage + system pref before hydration to avoid flash. CSS vars defined in `globals.css` under `.dark {}`.
- **Chart canvas backgrounds:** Must set `Chart.defaults.backgroundColor = 'transparent'` or dark mode cards show white chart backgrounds.
- **CSS specificity on glass-card:** `:root .glass-card` (light mode) beats `.glass-card` (base dark) because `:root` always matches `<html>`. Must use `:root:not(.dark) .glass-card` for light-only overrides. Fixed 2026-04-07.
- **Dev server:** Must use `--hostname 0.0.0.0 --webpack` (Turbopack hangs on non-localhost connections).
- **Env vars:** Missing `OPENROUTER_PROV_KEY` breaks API routes silently — no error page, just empty data.

## Active Work

**Stitch design pass (#15):** Production-grade visual polish. Bento grid sidebar, muted color palette, styled tables/budget bar/alerts/heatmap zone all done. Dark mode card bug fixed (2026-04-07). Remaining: general visual polish pass across all zones.

## Features

- Multi-zone dashboard (Dashboard / Breakdown / Heatmap)
- Real-time webhook ingestion from OpenRouter
- Telegram budget alerts
- App exclusion filters
- Date range picker with quick filters (1d, 7d, 30d)
- Budget thresholds with progress bar
- CSV upload fallback
