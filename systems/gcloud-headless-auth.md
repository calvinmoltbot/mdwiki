---
title: gcloud auth on the headless Mac Mini
tags: [gcloud, gcp, auth, mac-mini, headless, debugging]
created: 2026-05-20
updated: 2026-05-20
status: active
related:
  - ./mac-mini.md
  - ./firebase-functions-iam-gotcha.md
  - ../projects/golf.md
---

# gcloud auth on the headless Mac Mini

Running `gcloud auth login` on the Mini (no local browser, accessed over SSH) hits two distinct failures. Both are about the auth flow, not permissions.

## Symptom 1 — `Missing required parameter: redirect_uri`

Plain `gcloud auth login` tries to spin up a local browser + loopback redirect. On a headless box there's no browser, the loopback can't bind, and gcloud (SDK ~561) errors with:

```
ERROR: Missing required parameter: redirect_uri
```

**Fix**: use the no-launch-browser flow, which prints a URL to open elsewhere and prompts for a verification code instead of a loopback redirect:

```bash
gcloud auth login --no-launch-browser <account@gmail.com>
```

Open the printed URL in Chrome **on the MacBook**, sign in, approve, copy the verification code back into the prompt.

## Symptom 2 — `gcloud crashed (EOFError): EOF when reading a line`

The `--no-launch-browser` flow then waits at `Once finished, enter the verification code:`. If you run it through **Claude Code's `!` prefix**, that shell has no interactive stdin — gcloud gets EOF and crashes before you can paste the code.

```
Once finished, enter the verification code provided in your browser: ERROR: gcloud crashed (EOFError): EOF when reading a line
```

**Fix**: run `gcloud auth login --no-launch-browser` in a **real interactive terminal** (a separate iTerm2 tab SSH'd into the Mini), not via `!`. The verification code is PKCE-bound to that specific login session, so it can't be pasted into a *different* / fresh `gcloud` process — it must go into the same prompt that printed the URL.

## Picking the right active account

gcloud commands run as `core/account`. After logging in, point it at the account that actually has rights on the target project:

```bash
gcloud config set account <account@gmail.com>
gcloud auth list           # confirm the * is on the right account
```

Verify project access before relying on it:

```bash
gcloud projects get-iam-policy <project-id> \
  --flatten="bindings[].members" \
  --filter="bindings.members:<account@gmail.com>" \
  --format="value(bindings.role)"
```

## Production-deploy guardrail

Claude Code's auto-mode classifier **blocks production deploys** (e.g. `firebase deploy --only functions` to a live project) even after auth is fixed — authenticating to fix an error is not the same as approving the deploy. Expect to give explicit approval, or run the deploy yourself.

## Worked example (golf, 2026-05-20)

Deploying the golf Cloud Functions ([scorer-link IAM gotcha](./firebase-functions-iam-gotcha.md)) needs gcloud active as `calvin.orr@gmail.com` (owner on `holywoodgolf`; `calvinmoltbot@gmail.com` has **no** access). Sequence that worked:

1. In a real terminal: `gcloud auth login --no-launch-browser calvin.orr@gmail.com` → complete in MacBook Chrome
2. `gcloud config set account calvin.orr@gmail.com`
3. `pnpm --filter functions run deploy` (the `deploy-with-iam.sh` wrapper re-grants `allUsers` invoker after deploy)

## Related

- [Mac Mini setup](./mac-mini.md) — headless box, SSH from MacBook, no local browser.
- [Firebase callable functions IAM gotcha](./firebase-functions-iam-gotcha.md) — why the gcloud step is needed in the first place.
