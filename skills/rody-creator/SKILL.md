---
name: rody-creator
description: Use this skill when a creator-tier (non-staff) user is working with the @rodyssey/cli tool (`ro`) to build and ship their own webapp against the rodyssey CMS — creating your app, deploying it, configuring its metadata, and going live via the share ladder. Trigger on `ro auth login`, `ro auth me`, `ro auth logout`, `ro app create`, `ro app deploy`, `ro app config *`, `ro app assets push`, or `ro app share *`; on requests like "create my app", "go live", "publish my app", "share this with my family/school/everyone"; on questions about `webapp.config.json`, `VIBED_APP` drafts, the share ladder (private/family/school/public), or CMS errors like `not_eligible`, `share_rung_forbidden`, `config_key_forbidden`; on projects whose `.env` holds a production `WEBAPP_ID`/`DEPLOY_TOKEN` for a personal or school-owned app. This is the creator/public tier — for RO staff workflows (promote, global-config, entity groups, GameSDK actions), use the `rodyssey-cli` skill instead.
---

# Rodyssey CLI — Creator Edition

`@rodyssey/cli` (`ro`) scaffolds, configures, and ships your own webapp to the rodyssey CMS. This edition covers the **creator tier**: apps you personally own (or, with `--school`, that your school owns), **production only**. There is no dev/staging tier and no `promote` step — going live means climbing the **share ladder**.

**Contents:** Command map · Two credentials, not one · The one rule: always `-e production` · Getting started, start to finish · Auth: login, me, logout · `app create` · `app deploy` · `webapp.config.json` and `app config` · `app assets push` · The share ladder (`app share`) · Typed errors · Headless / non-TTY · Common pitfalls

## Command map

```
ro auth login    -e production [--no-open] [--callback-port <p>]
ro auth me       -e production [--remote]
ro auth logout   -e production [--all]

ro app create <name> [--school]
                      # production, the SPA template, and --auto are all automatic

ro app deploy    -e production [--push-config]

ro app config get   -e production [--webapp-id <id>] [--out <file>]
ro app config set   -e production [--webapp-id <id>] [--title ...] [--description ...]
                                  [--cover-img ...] [--localization <json|file>] [--details <json|file>]
                                  [--dry-run]
ro app config pull  -e production [--webapp-id <id>] [--out <file>]
ro app config push  -e production [--webapp-id <id>] [--file <path>] [--dry-run] [-y|--yes]

ro app assets push <paths...> -e production [--dest <dir>] [--dry-run] [--json]

ro app share [private|family|school|public] -e production [--webapp-id <id>]
             [-y|--yes] [--dry-run] [--json]
```

## Two credentials, not one

- **Login session token** — from `ro auth login -e production`, stored in `~/.rodyssey/config.json`. Powers `auth me`, `auth logout`, `app create` (used both to detect you're creator-tier and to authorize the create request itself — there's no deploy token yet at create time), and `app share`.
- **Per-app deploy token** — `DEPLOY_TOKEN`, written into the project's `.env` by `app create` once the webapp is provisioned. Powers `app deploy`, `app config get/set/pull/push`, and `app assets push`.

They're independent: `ro auth logout` only revokes the login session token — it has no effect on any existing project's `.env`, which keeps deploying/configuring/pushing assets fine. Conversely, losing/regenerating a project's `.env` doesn't touch your login session.

## What you may run unasked

The user's request authorizes the actions it names and the setup those actions
require. **It never authorizes shipping.**

- **Always fine — read-only:** `auth me`, `app config get`, `app share` with no
  argument (status), `app assets push --dry-run`, any `--dry-run`.
- **Authorized by "build me an app":** `ro app create`. The CMS webapp has to
  exist before the project can run at all — the GameSDK needs a real
  `WEBAPP_ID` in its iframe — so provisioning is part of setting the project
  up, not part of publishing it. Local work is likewise yours: editing files,
  installing dependencies, running and testing the app.
- **Stop and ask, every time:** `app deploy`, `app config set`/`config push`,
  `app assets push`, `app share <rung>`, `app release`. Build the app, get it
  working locally, then ask one question: *"It's ready — want me to deploy it?"*

**`-y`/`--yes` suppresses the CLI's own confirmation prompt. It is not the
user's authorization.** It exists so an action the user already asked for
doesn't hang in a non-TTY. Never reach for it to skip asking.

| Rationalization | Reality |
|---|---|
| "Deploy is safe — the app stays private until it's shared" | Only true for a draft nobody has shared yet. On an app already at `family`, `school`, or `public`, **deploy ships your new code straight to those viewers** — no further command, no review. You usually can't tell which case you're in without checking. |
| "They said build it and get it working — deploying is what 'working' means" | It works when it runs locally. Deploying is a separate decision that belongs to the user. |
| "They said don't ask, just do it / I trust you" | That's a mood, not authorization for a named action. Do all the building, then ask the one question. |
| "The config change is pointless unless I push it" | Show the `--dry-run` diff and ask. |
| "It's reversible — I can downgrade or redeploy after" | Reversible isn't the bar. *Asked* is. |

**Red flags — stop:** you're about to type `-y` on a command the user never
named; you're about to call the task "done" by deploying; you caught yourself
thinking "low blast radius".

## The one rule: always pass `-e production`

Every command above defaults to `-e development` if you omit `-e` — and creator accounts have no development session or webapp. Forgetting it fails differently per command, so don't assume they all fail the same way:
- **`ro auth me`** throws ``No local CMS session found for [development]. Run `ro auth login --env development` first.``
- **`ro app share`** throws a differently-worded (but same-idea) error: ``No CMS auth token found for [development]. Run `ro auth login --env development` first.``
- **`ro auth logout`** does **not** throw — it just prints `No stored session for [development].` and exits cleanly (nothing to revoke, nothing to forget).
- **Deploy-token commands** (`app deploy`, `app config get/set/pull/push`, `app assets push`) read `DEPLOY_TOKEN` from `.env` regardless of `-e`, but send it to the *development* CMS host — which won't recognize a production token, so the request fails.

All of the above only happen because you only ever logged in with `-e production` — there's no stored session under `development` to find.

**`ro app create` is the one exception** — it detects a stored creator-tier session and forces production for you automatically, printing an info line (`ℹ️  Creator accounts work directly against production — apps start as private drafts.`) when it does. This only works if you're already logged in (`ro auth login -e production`) — without a stored session, `create` can't tell you're creator-tier and won't apply the defaults.

## Getting started, start to finish

```bash
# 1. Log in once per machine (skip if `ro auth me -e production` already shows a session)
ro auth login -e production        # opens the web-client's sign-in + consent page in your browser (3 scopes)

# 2. Create the app — production + the SPA template + --auto are automatic
ro app create wwii-explorer
cd wwii-explorer

# 3. Build it — normal template dev loop
bun install
bun run dev   # blocking, interactive dev server — for humans; headless agents should skip this and go to step 5 (`ro app deploy` runs its own production build)

# 4. Set metadata: edit webapp.config.json (title, description, config), then:
ro app config push -e production --dry-run   # preview the delta
ro app config push -e production -y          # apply (drop -y if you want the interactive Y/N prompt)

#    Cover image goes through assets, then into the config file:
ro app assets push ./cover.png --dest images -e production --json   # → public URL
#    put that URL into webapp.config.json's "coverImg", then:
ro app config push -e production -y

# 5. Ship it — live in production, visible only to you until you share it
ro app deploy -e production

# 6. Go live, one rung at a time
ro app share family -e production -y     # instant
ro app share school -e production -y     # needs a teacher/school-admin role at your school
ro app share public -e production -y     # files a review request — staff approve it
```

## Auth: login, me, logout

- **`ro auth login -e production`** — browser PKCE flow. The browser opens the **web-client's own sign-in and consent page** — the same site you already use, not a separate admin tool — at `https://app.rodyssey.ai/auth/cli-authorize`. If you're not already signed into the web-client, you'll see its normal login first and land on the consent screen right after. The consent screen shows exactly three scopes for creator accounts: `webapps:create`, `webapps:deploy-token:create`, `webapps:config`. On success it prints `✅ Logged in to CMS [production]` and the CMS URL. Headless/remote sessions: add `--no-open` to print the URL instead of opening a browser, and relay it to a human to complete in any browser — there is no fully non-interactive login. `--callback-port <p>` pins the local callback port (useful over SSH port-forwarding; default is a random free port). The command itself **blocks and doesn't exit** until the human finishes the consent screen and the local callback lands (default timeout 5 minutes, `--timeout <seconds>` to change it) — so for a headless agent the flow is just: run it with `--no-open`, relay the URL, then wait for the command to exit on its own (no polling needed), and run `ro auth me -e production` afterward as confirmation.
  - **If the account isn't authorized for the CLI at all** (not a `cli-creators` member and not staff), the consent page redirects back before showing any scopes, and the CLI prints `Error: CMS login failed: access_denied: Your account is not enabled for the Rodyssey CLI. Ask an RO admin to add you to the cli-creators group.` That's an eligibility problem, not a CLI bug — ask an RO admin to add the account to the `cli-creators` group, then try again.
  - **Landed on the wrong account's consent screen?** This page has no in-page "switch account" link — if you're already signed into the web-client as someone else, sign out of the web-client entirely (not just close the tab) and re-run `ro auth login`. (Advanced/rare: `--consent-url <url>` or `RO_CONSENT_URL` can point the browser at a different web-client host — not needed for normal use, since `-e production` already resolves the right one.)
- **`ro auth me -e production`** — prints `Logged in as: <email>`, `Granted scopes: webapps:create, webapps:deploy-token:create, webapps:config`, and — if your account is tied to a school — `Acting for school: <name> (id: <id>)`. That acting-school line is exactly what `ro app create --school` and `ro app share school` check against. Add `--remote` for a server-side freshness check.
- **`ro auth logout -e production`** (`--all` for every stored environment) — revokes the session server-side (so it can't be reused even if the local file leaks) and forgets it locally. If the server call fails (offline, already revoked), it still forgets locally and prints a warning that server-side revocation didn't happen.

## `app create` — production, SPA, automatic

```bash
ro app create <name>              # your own app
ro app create <name> --school     # school-owned app: ownerSchoolId = your acting school
```

- No `--template` flag needed — creator accounts always get the SPA template (the fullstack/Workers template is staff-only; explicitly passing `--template webapp-fullstack` fails fast with `Fullstack template requires staff access.`).
- No `--auto` flag needed — it's implied. The CLI provisions the CMS webapp, writes `WEBAPP_ID`/`DEPLOY_TOKEN` to `.env`, and pulls a starting `webapp.config.json`.
- The app is created as a **private draft** (`VIBED_APP`, share mode `private`) — only you can see it until you run `ro app share`.
- **`--school`** requires you to currently hold a teacher or school-admin role at your acting school (see `ro auth me`). Without it, the server rejects with `share_rung_forbidden` and a self-contained message — see "Typed errors" below. `createdBy` still records you as the individual creator either way.
- `-e` doesn't matter here — omit it or pass `-e production`, both work; a non-production value is corrected automatically with an info line (see "The one rule" above).
- Expect a few `ℹ️` info lines before the create output starts, confirming the defaults it applied on your behalf — e.g. `ℹ️  Creator accounts work directly against production — apps start as private drafts.` and `ℹ️  Using the webapp template.` (and, since you didn't pass `--auto`, `ℹ️  Provisioning automatically (creator accounts always use --auto).`). These are informational, not errors. Then the normal create output: `🌐 Creating CMS webapp in [production]...`, `🔐 Wrote WEBAPP_ID and DEPLOY_TOKEN to .env`, `📍 Webapp ID: ...`, `📄 Wrote webapp.config.json (pulled from CMS)`, and finally `✅ Project "<name>" created successfully!` with the `cd`/`bun install`/`bun run dev` next steps.
- **Log in first.** The CLI only applies all of the above by finding a stored production session (`ro auth login -e production`) for the identity check. Skip that, and `ro app create <name>` (without an explicit `--auto`) does **not** error — it silently falls back to a bare scaffold: clones the template locally with no CMS webapp, no `.env`, and an empty `webapp.config.json`. If `.env`/`WEBAPP_ID` are missing after create, this is why — log in and re-create.

## `app deploy`

```bash
ro app deploy -e production
```

Auto-detects the SPA build pipeline (`npm run build` → upload assets in batches → zip + upload HTML → trigger the CMS deploy → sync API/cron scripts) and prints the public app URL when it finishes. After a successful deploy: if `webapp.config.json` differs from the CMS you're prompted to push it (or it auto-pushes with `--push-config`); if the file is simply missing, the CLI pulls it for you automatically, no prompt.

## `webapp.config.json` and `app config`

Same committed-config flow as the full CLI:

- **`ro app config pull -e production`** — GETs the CMS config and writes a clean `webapp.config.json` (the `{ webappId, updatedAt }` envelope stripped, empty defaults like `tags: []` removed).
- **`ro app config push -e production`** — reads `webapp.config.json`, previews a New/Update/Delete diff against the CMS, prompts to confirm (`-y` to skip, `--dry-run` to preview only), then PATCHes. Only keys present in the file are touched; clear a field with an explicit `null`.
- **`ro app config set -e production --title ... --description ... --cover-img ...`** — one-off field updates without touching the file.

**What you can write** (allowlisted for creator-tier apps): the branding fields — `title`, `description`, `coverImg`, `localization`, `widgetManifest` — and the free-form `config` runtime blob. Everything else — `tags` (including age-rating tags), `characterIds`, `knowledgeIds`, `imageStyleIds`, `giftBoxConfig`, and the flashcard fields — is rejected with `config_key_forbidden` naming the key. Those stay staff-curated: content-ID fields reference CMS entities you can't enumerate, and age rating is assigned during the `public` review step, not set by hand.

Use **`coverImg`** for the cover image. There is no `coverUrl` field for creator apps — don't invent one; only `coverImg` is ever accepted.

⚠️ Never seed `webapp.config.json` from `ro app config get --out` — only `pull` strips the CMS's empty defaults. Feeding a raw `get` response into `push` clears real fields (they read as explicit clear-signals).

## `app assets push` — cover images and other files

```bash
ro app assets push ./cover.png --dest images -e production --json
# → { "success": true, "assets": [{ "path": "images/cover.png", "url": "https://…", "size": 12345 }] }
ro app config set -e production --cover-img "<url>"
# — or put the URL in webapp.config.json's coverImg and `ro app config push -e production`
```

Re-pushing a path **overwrites** it (the public cache is 5 minutes — prefer a new filename over reusing a path that's already referenced, so users don't briefly see the stale cached version). No confirmation prompt (machine-to-machine, like `app deploy`); use `--dry-run` to preview the local→remote mapping.

## The share ladder — `app share`

Your app starts **private**. Going live means climbing four rungs: `private → family → school → public`.

**Rungs are directly addressable, not a required sequence.** `ro app share <rung>` always requests that exact rung — there's no client- or server-side requirement to pass through the ones in between. Going straight from `private` (or `family`) to `public` is completely normal. `school` only matters if you actually want school-level visibility and hold the role for it; if you don't, you simply never use it — jump from `family` straight to `public` instead.

```bash
ro app share -e production                     # print the current rung (+ pending review, if any)
ro app share family -e production -y            # instant if you're your linked family's billing holder
ro app share school -e production -y            # instant IF you hold teacher/school-admin at your acting school
ro app share public -e production -y            # files a review request — not applied yet
ro app share private -e production -y           # downgrade — always allowed
```

Any call that names a rung first prints a banner — `Change visibility of <webappId> to [<rung>] on [production]` — before doing anything else; without `-y`/`--yes` on a TTY it then asks `<same banner>? (y/N):`. On success (any rung except a pending `public` request) it prints `✅ Share mode set to [<rung>].`.

- **`family`** — applies immediately, owner self-serve, **provided you're the primary/billing account holder of an RO family linked to your account**. Makes the app visible to your RO family: the household linked to your account. Family members see it on their RO home. If there's no linked family, or you're not its billing holder, the server rejects with `share_rung_forbidden` and the CLI appends `Family sharing requires an RO family linked to your account (you must be the family's primary/billing account holder).` — see "Typed errors" below.
- **`school`** — applies immediately, but only if you currently hold a teacher or school-admin role at the acting school on your session (`ro auth me` shows this as `Acting for school: ...`). The server re-checks your role at request time, not just at login — if your role changed since you logged in, re-run `ro auth login -e production`.
- **`public`** — never applies immediately, no matter who requests it. The response is a pending review (HTTP 202); the CLI prints ``⏳ Public review requested — status: pending. Check later with `ro app share`.``. A staff reviewer approves (or denies) the request out of band and assigns the age rating — there is nothing further to do from the CLI. Check back any time with `ro app share -e production`: a pending review shows as `⏳ Public review pending (requested <date>)` beneath the current rung.
- **Downgrades** (moving back down the ladder, including all the way to `private`) are always owner self-serve, no role check.
- Running it with **no argument** never changes anything — it only prints the current rung and any pending review.
- **Non-TTY:** without `-y`/`--yes` the command errors out — pass `--yes` or `--dry-run` in scripts/agents.

## Typed errors

The CLI surfaces certain server error codes with extra context. The exact shape differs by which command hit it — verified against source, don't assume they're identical:

| Code | Where you'll see it | What the CLI actually prints |
|---|---|---|
| `not_eligible` | `ro app create` | One line, the server's message verbatim, e.g. `Your account is not enabled for CLI app creation. Ask an RO admin to add you to the cli-creators group.` |
| `share_rung_forbidden` | `ro app create --school` | One line, verbatim, e.g. `School-owned apps require a teacher or school-admin role at your acting school.` |
| `share_rung_forbidden` | `ro app share school` | Three lines: `POST <url> failed: 403 Forbidden`, the raw JSON error body, then `Raising to school visibility requires a teacher or school-admin role at your acting school (see ro auth me).` |
| `share_rung_forbidden` | `ro app share family` (no RO family linked to your account, or you're not its billing holder) | Three lines: the same diagnostic + JSON shape, then `Family sharing requires an RO family linked to your account (you must be the family's primary/billing account holder).` — the CLI picks this remediation over the school one based on which rung you requested, since the server reuses the same error code for both gates. |
| `share_review_pending` | `ro app share public` (while a review is already pending) | Three lines: the same diagnostic + JSON shape, then `A public review is already pending.` |
| `not_owner` | any `ro app share` call against a webapp you don't own or manage | Two lines: diagnostic + raw JSON error body, no extra remediation line — double-check `--webapp-id` / `WEBAPP_ID`. |
| `config_key_forbidden` | `ro app config set` / `ro app config push` | Two lines: `Config update failed: <status> <statusText>`, then the server's raw error body naming the rejected key. Remove that key from `webapp.config.json` (or the field flag) and retry. |
| `apptype_locked` | only if something runs `ro app release` directly — you shouldn't need to | Three lines: diagnostic, the server's message, then ``→ Use `ro app share` to manage this app's visibility.`` — a creator-tier app's visibility is managed by the ladder, not by changing its app type. |

In every case the CLI's top-level error handler prints `Error: <the lines above>` and exits 1 — set `RO_DEBUG=1` to see a full stack if you're chasing a CLI-internal bug rather than a server rejection.

## Headless / non-TTY

| Command | Non-TTY without `-y`/`--yes` |
|---|---|
| `app create` | **safe only if a creator-tier production session is already stored** (see `ro auth me -e production` below) — then it's prompt-free, no confirmation asked. **Without a detected session, it's NOT safe**: `resolveCreatePlan` returns no default template, so — unless you passed `--template` yourself — the CLI falls through to an interactive picker (`selectTemplate()`, plain `readline`, no TTY check) and **hangs forever** with no error, no timeout. |
| `app config push` | errors out — pass `--yes` or `--dry-run` |
| `app share <mode>` | errors out — pass `--yes` or `--dry-run` |
| `app deploy` config-drift check | warns only if `webapp.config.json` differs from the CMS (auto-push instead with `--push-config`); a missing file is auto-pulled silently either way |
| `app config set` / `app assets push` | prompt-free by design (deploy-token, machine-to-machine) |

`ro auth login` is never headless — the browser consent screen always needs a human; `--no-open` only skips auto-opening the browser, it still prints the URL for a human to complete elsewhere.

**Pre-flight for any headless/agent run that includes `app create`:** run `ro auth me -e production` first and confirm it shows a session. If it doesn't, log in (`ro auth login -e production`) before calling `create` — don't rely on `create` to fail loudly if the session is missing; for the template picker specifically, it won't fail at all, it hangs.

## Common pitfalls

- **"No CMS auth token found for [development]" / a request that mysteriously fails** — you forgot `-e production` on a command other than `app create`. See "The one rule" above.
- **`app create` silently produces a broken project if you skip logging in first** — without a detected creator-tier session, `create` never implies `--auto`, no matter what: in a **headless** run without an explicit `--template`, it hangs forever on an interactive picker (see "Headless / non-TTY" above); in a **TTY** run, or whenever `--template` was passed, it instead completes normally-looking but with **no CMS webapp, no `.env`, and an empty `webapp.config.json`** — no error, no warning, either way. Answering the interactive picker prompt does not rescue you into full provisioning. Always run `ro auth me -e production` (and log in if needed) before `create`, and check `.env`/`WEBAPP_ID` exist right after.
- **`config push` wiped fields** — you seeded `webapp.config.json` from `ro app config get --out` instead of `ro app config pull`. Only `pull` strips the CMS's empty defaults (`tags: []`, etc.); pushing those back is treated as a real clear signal. Re-seed with `pull`.
- **`config_key_forbidden` on push/set** — you (or a stale config file) included a staff-curated key — `tags`, `characterIds`, `knowledgeIds`, `imageStyleIds`, `giftBoxConfig`, or a flashcard field. Remove that key; only the branding fields and `config` are writable on creator-tier apps (see "What you can write" above).
- **`ro app release` doesn't work for you** — that's expected; creator-tier apps (`VIBED_APP`) reject it with `apptype_locked`. Use `ro app share` to change visibility instead.
- **Cover image not showing after a re-push** — the public asset cache is 5 minutes; prefer a new filename when replacing an already-referenced image rather than waiting it out.
- **`ro app share public` seems stuck** — it's supposed to: the rung only changes once a staff reviewer approves the request. `ro app share -e production` shows the pending status; there's no CLI action that speeds this up.
- **`ro auth login` shows a consent screen for the wrong person** — you (or whoever's at the keyboard) is already signed into the web-client as someone else. There's no switch-account link on this screen — sign all the way out of the web-client and re-run the login command. See "Auth: login, me, logout" above.
