---
title: Vercel deploy gotchas (the silent failures)
tags: [vercel, deployment, debugging, infrastructure]
created: 2026-04-25
updated: 2026-04-25
status: active
related:
  - ./vercel-neon-marketplace.md
  - ../projects/letterboxd-tracker.md
---

# Vercel deploy gotchas — the silent failures

Vercel's deploy step (the bit *after* "Build Completed") can fail in ways that surface **no error message anywhere** — not in build logs, not in the dashboard banner, not in the deployment API, not via the CLI. Build is green, deploy is red, and you have nothing to go on.

This page exists because we lost ~6 hours debugging exactly this on `letterboxd-tracker` on 2026-04-25, and tried (and ruled out) a long list of plausible-but-wrong theories before bisecting found the cause. If you hit the same pattern, check this page **first**.

## The signature

- ✅ Build logs end cleanly with `Build Completed in /vercel/output [Xs]` then `Deploying outputs...`
- ❌ Deployment immediately flips to ERROR with no further log lines
- ❌ Dashboard shows generic "Your deployment has failed. Please check out the logs." (no specific error)
- ❌ Deployment API returns no `errorMessage` / `errorCode`
- ❌ `vercel inspect` shows nothing useful
- ❌ `vercel logs` only streams runtime logs (nothing about the deploy step)

## Cause #1 — `.npmrc` with `${HOME}` paths

The actual root cause of the letterboxd-tracker debacle. A `.npmrc` checked into the repo containing:

```
store-dir=${HOME}/.cache/letterboxd-tracker-pnpm-store
virtual-store-dir=${HOME}/.cache/letterboxd-tracker-pnpm-virtual-store
```

On Vercel's build container, `${HOME}` resolves to `/vercel/`, which is **outside the project root** `/vercel/path0`. pnpm therefore installs `node_modules` outside the project tree. `next build` succeeds because symlinks resolve at build time. But Vercel's post-build packaging of serverless function bundles fails when the bundle's `node_modules` symlinks point outside `/vercel/path0`.

**Fix:** Delete the `.npmrc`, or use a project-relative path:

```
store-dir=./.pnpm-store
```

(and add `.pnpm-store` to `.gitignore`)

**Rule:** Never use `${HOME}` or any out-of-tree absolute path in any tool config (`.npmrc`, `.yarnrc`, `.pnpmfile.cjs`, etc.) checked into a repo deployed to a serverless platform. The deploy container's `$HOME` is not your dev machine's `$HOME`.

## Cause #2 — Vercel CLI v51 silently storing empty values for sensitive env vars

Different gotcha, related debugging — once deploys went green, all DB-touching endpoints 500'd because `DATABASE_URL` was empty in production.

When scripting `vercel env add KEY production --value "..." --yes` (or piping via stdin), CLI v51 silently stores the value as `""` because production/preview default new entries to "sensitive" mode.

**Fix:** Always pass `--no-sensitive` when scripting prod env adds, and verify with a pull-back:

```bash
echo "$VAL" | vercel env add KEY production --yes --no-sensitive
vercel env pull /tmp/check --environment=production
grep "^KEY" /tmp/check  # should show the real value, not ""
```

If the value genuinely needs to be sensitive (write-only), add it via the dashboard UI — don't trust the CLI for it in this version.

## Things to rule out *before* digging deeper

When you hit the silent-deploy-failure pattern, check these in order — they're all things we eliminated on letterboxd-tracker that turned out NOT to be the cause:

1. **`.npmrc` with `${HOME}`** — see Cause #1
2. **Function count** — Vercel auto-dedupes Next.js routes via symlinks; check actual function count via `vercel build && ls .vercel/output/functions/api/*.func`. Hobby's "12 function" limit applies to the *deduped* count, not the route count.
3. **Region config** — On Hobby, `regions` is restricted to your account's home region; on Pro you can pick freely. A bad region setting will sometimes silently fail.
4. **Broken env var references** — Old env vars (created via `@secret-name` style references when Vercel still had Secrets) appear as "Needs Attention" in the dashboard. CLI shows them as "Encrypted" but they don't resolve at runtime. Delete and re-add.
5. **AUP / abuse flags** — If your code ships URLs of known piracy sites or other AUP-violating content as literal strings in serverless function bundles, Vercel's automated abuse detection may silently reject the deploy. The repo/team flag persists across project recreations. Even though this *wasn't* the letterboxd-tracker cause, it remains plausible enough to warrant checking — keep piracy-adjacent code out of Vercel bundles regardless (run on Mini, proxy via Tailscale).

## How to diagnose when the API gives you nothing

The bisect approach worked on letterboxd-tracker. Don't burn hours guessing — bisect:

```bash
# In a /tmp clone, link to the project, deploy preview from a suspect commit
cd /tmp && git clone <repo> bisect && cd bisect
git checkout <last-known-good-sha>
mkdir -p .vercel
cat > .vercel/project.json <<EOF
{"projectId":"prj_...","orgId":"team_...","projectName":"...","settings":{"framework":"nextjs"}}
EOF
vercel deploy --scope <team> --yes
# Wait for Ready/Error, then bisect to next commit
```

Preview deploys are cheap and surface failures as fast as production. ~5 deploys = root cause for any reasonably-sized history.

## See also

- `letterboxd-tracker` 2026-04-25 incident — the originating debugging session for this page
- Vercel Acceptable Use Policy: https://vercel.com/legal/acceptable-use-policy
