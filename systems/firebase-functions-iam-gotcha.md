---
title: Firebase Callable Functions — public invoker IAM gotcha
tags: [firebase, cloud-functions, iam, gcp, debugging]
created: 2026-05-18
updated: 2026-05-20
status: active
related:
  - ../projects/golf.md
  - ./gcloud-headless-auth.md
---

# Firebase Callable Functions — public invoker IAM gotcha

**Symptom**: a Firebase Cloud Function callable that should be invokable from an unauthenticated client (e.g. a public magic-link page) returns:

- Browser console: `FirebaseError: internal` (with no helpful message)
- OPTIONS preflight to `https://us-central1-{project}.cloudfunctions.net/{name}` returns **403 Forbidden**
- Direct `curl` to the function URL also returns 403

Looks like a CORS / function code bug. **It isn't.** The platform is rejecting the request *before* your function ever runs.

## Root cause

Since ~2024 GCP changed the default for new Cloud Functions: they're created as IAM-protected, with no `allUsers` invoker permission. `firebase deploy --only functions:X` does not auto-grant `allUsers` invoker on (re)create. So:

- The function deploys successfully
- Adding `console.error` / debug logging in the function makes no difference (the function isn't running)
- Wrapping the handler in try/catch makes no difference (same reason)
- Error code is always `internal`, error message is always `internal` — that's the platform's default for an unauth'd reject

## Fix

```bash
gcloud functions add-iam-policy-binding <function-name> \
  --region=us-central1 --project=<project-id> \
  --member="allUsers" --role="roles/cloudfunctions.invoker"
```

Apply to every callable that needs unauth access. Verify with:

```bash
gcloud functions get-iam-policy <function-name> --region=us-central1 --project=<project-id>
```

Should list `allUsers` under `roles/cloudfunctions.invoker`.

## Permissions to run the fix

The `firebase-adminsdk-fbsvc@<project>.iam.gserviceaccount.com` service account does **not** have `cloudfunctions.functions.setIamPolicy`. The fix needs a human running `gcloud` interactively as an account with project rights (on the headless Mini this means the [no-launch-browser auth flow](./gcloud-headless-auth.md) — `gcloud auth login` fails outright there) OR the binding applied via the GCP Console UI:

1. https://console.cloud.google.com/functions/list?project=<project-id>
2. Click the function → **Permissions** tab → **Grant access**
3. Principal: `allUsers`, Role: **Cloud Functions Invoker** → Save → confirm the public-access warning

Two clicks per function.

## Persistence

The binding **persists across function updates** (`firebase deploy --only functions:X` after the first time keeps it). It is stripped if:

- The function is deleted and recreated (e.g. switching between 1st and 2nd gen)
- Anyone explicitly removes the binding via console / CLI
- A new deployment is the function's first deployment in a new region

So in practice you set it once per function and forget — until someone redeploys to a fresh function name or migrates gen.

## How to recognise this in the wild

If you see all three of these together, this is almost certainly the bug:

- Client gets `FirebaseError: internal` with no useful message
- Server-side logs in Firebase / Cloud Logging show nothing — no invocations recorded
- `curl https://us-central1-{project}.cloudfunctions.net/{name}` returns plain HTTP 403 with no Firebase error envelope

Don't bother adding error logging to the function code. Don't bother redeploying. Check the IAM binding first.

## Background

The function in question doesn't have to use `request.auth` — callable functions can validate authorization via their own token system (e.g. magic-link tokens stored in Firestore). But the *invocation itself* still needs IAM permission to call the function at all. The platform-level IAM check happens before the function code runs and before any auth claims are passed to it.

## Related

- [Golf project — scorer link architecture](../projects/golf.md): primary use case in our stack. The scorer admin is unauth by design; the security comes from the token, validated server-side. We deploy `submitScores` and `getScorerRoundContext` as callables; both need this binding.
- Issue [calvinmoltbot/golf#27](https://github.com/calvinmoltbot/golf/issues/27): post-deploy wrapper script — **now exists** as `functions/scripts/deploy-with-iam.sh` (run via `pnpm --filter functions run deploy`); re-grants `allUsers` invoker on `getScorerRoundContext`, `submitScores`, `getPublicShareContext` after every deploy.
- [gcloud auth on the headless Mac Mini](./gcloud-headless-auth.md): how to get gcloud authenticated to run the binding/deploy from the Mini.
