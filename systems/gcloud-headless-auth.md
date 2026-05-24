---
title: gcloud auth on the headless Mac Mini
tags: [gcloud, gcp, auth, mac-mini, headless, debugging]
created: 2026-05-20
updated: 2026-05-24
status: active
related:
  - ./mac-mini.md
  - ./firebase-functions-iam-gotcha.md
  - ./nextjs-tailscale-dev-origins.md
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

## ADC for a *library* (no gcloud) — Claude-driven loopback OAuth

The above is about authenticating the **gcloud CLI** itself. A different, harder
case: a **library/app** that needs **Application Default Credentials** (ADC) with a
**sensitive scope** (e.g. the GA4 Data API needs `analytics.readonly`) on the
headless Mini. Here `gcloud` is the wrong tool entirely, and its headless flows
dead-end:

- `gcloud auth application-default login --scopes=…analytics.readonly` with the
  **default** client → *"This app is blocked"* (analytics.readonly is sensitive on
  personal Gmail).
- Use your **own** Desktop OAuth client (`--client-id-file`) to bypass that, and
  `--no-launch-browser` is **rejected**; you must use `--no-browser`, which prints
  a `--remote-bootstrap` command that has to run on a machine with **both a browser
  and gcloud** — the MacBook has Chrome but **no gcloud**. Dead end.

**Workaround — drive the OAuth flow manually, no gcloud:** Claude can run the whole
loopback flow itself using Chrome automation on the MacBook.

1. **Prereqs (one-time):** a **Desktop** OAuth client in the GCP project
   (`redirect_uri=http://localhost`); consent screen External/**Testing** with the
   user added as a **test user** (test users skip sensitive-scope verification).
2. **Build the consent URL** for that client:
   `https://accounts.google.com/o/oauth2/auth?client_id=…&redirect_uri=http://localhost&response_type=code&scope=<scope>&access_type=offline&prompt=consent`
3. **Drive MacBook Chrome** to it; the **user clicks through their own consent**
   (Advanced → "Go to … (unsafe)" for the Testing-mode warning).
4. Google redirects to `http://localhost/?code=…`. The page fails to load (nothing
   is listening on the MacBook's port 80) **but the address bar keeps the URL** —
   read the `?code=` from it. No local listener needed.
5. **Exchange the code on the Mini** (`POST https://oauth2.googleapis.com/token`
   with the client id/secret + `redirect_uri=http://localhost`). Codes are
   **single-use** — capture + exchange in one shot.
6. **Write the ADC file** `~/.config/gcloud/application_default_credentials.json`
   as an `authorized_user` doc: `client_id`, `client_secret`, `refresh_token`,
   `type: "authorized_user"`, `quota_project_id: <project>`. The Google client
   libraries pick it up ambiently (leave `GOOGLE_APPLICATION_CREDENTIALS` unset).

Gotchas: a **stale** ADC file (e.g. from an old default-client login with no
analytics scope) silently shadows everything → `PERMISSION_DENIED: insufficient
authentication scopes`; back it up and overwrite. Service-account keys *authenticate*
but GA4 **rejects adding an SA email as a property user** ("doesn't match a Google
Account"), which is why the property owner's user ADC is used instead.

Worked end-to-end for hdwshopify GA4 (2026-05-24): property `538465497`, client in
project `hdw-analytics`. This is the general recipe whenever a headless-Mini app
needs ADC/OAuth and `gcloud`'s flows won't cooperate.

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
