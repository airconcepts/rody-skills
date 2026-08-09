---
name: rody-creator
description: Use this skill when a creator-tier (non-staff) user is working with the @rodyssey/cli tool (`ro`) to build and ship their own webapp against the rodyssey CMS — scaffolding your app yourself, provisioning it with `ro app init`, deploying it, configuring its metadata, and going live via the share ladder. Trigger on `ro auth login`, `ro auth me`, `ro auth logout`, `ro app init`, `ro app deploy`, `ro app config *`, `ro app assets push`, or `ro app share *`; on requests like "create my app", "scaffold my app", "go live", "publish my app", "share this with my family/school/everyone"; on the `/new-ro-app` guided flow; on questions about `webapp.config.json`, `VIBED_APP` drafts, the share ladder (private/family/school/public), or CMS errors like `not_eligible`, `share_rung_forbidden`, `config_key_forbidden`; on projects whose `.env` holds a production `WEBAPP_ID`/`DEPLOY_TOKEN` for a personal or school-owned app. This is the creator/public tier — for RO staff workflows (create-from-template, promote, global-config, entity groups, GameSDK actions), use the `rodyssey-cli` skill instead.
---

# Rodyssey CLI — Creator Edition

`@rodyssey/cli` (`ro`) provisions, configures, and ships a webapp **you scaffolded yourself** to the rodyssey CMS — creators never clone a template. This edition covers the **creator tier**: apps you personally own (or, with `--school`, that your school owns), **production only**. There is no dev/staging tier and no `promote` step — going live means climbing the **share ladder**.

**Contents:** Command map · Two credentials, not one · Environments: production, automatically · Getting started, start to finish · Auth: login, me, logout · Starting a new app (no clone) · `app deploy` · `webapp.config.json` and `app config` · `app assets push` · The share ladder (`app share`) · Typed errors · Headless / non-TTY · Common pitfalls

## Command map

```
ro auth login    [--no-open] [--callback-port <p>]
ro auth me       [--remote]
ro auth logout   [--all]

ro app init --title "<title>" [--school] [--with-sdk-files]
                      # run inside YOUR scaffolded project folder — provisions the CMS webapp,
                      # writes WEBAPP_ID/DEPLOY_TOKEN to .env, installs the game-sdk docs

ro app deploy    [--push-config] [--dist-dir <dir>] [--build-command <cmd>]

ro app config get   [--webapp-id <id>] [--out <file>]
ro app config set   [--webapp-id <id>] [--title ...] [--description ...]
                    [--cover-img ...] [--localization <json|file>] [--details <json|file>]
                    [--dry-run]
ro app config pull  [--webapp-id <id>] [--out <file>]
ro app config push  [--webapp-id <id>] [--file <path>] [--dry-run] [-y|--yes]

ro app assets push <paths...> [--dest <dir>] [--dry-run] [--json]

ro app share [private|family|school|public] [--webapp-id <id>]
             [-y|--yes] [--dry-run] [--json]
```

## Two credentials, not one

- **Login session token** — from `ro auth login`, stored in `~/.rodyssey/config.json`. Powers `auth me`, `auth logout`, `app init` (used both to detect you're creator-tier and to authorize the provisioning request itself — there's no deploy token yet at init time), and `app share`.
- **Per-app deploy token** — `DEPLOY_TOKEN`, written into the project's `.env` by `app init` once the webapp is provisioned. Powers `app deploy`, `app config get/set/pull/push`, and `app assets push`.

They're independent: `ro auth logout` only revokes the login session token — it has no effect on any existing project's `.env`, which keeps deploying/configuring/pushing assets fine. Conversely, losing/regenerating a project's `.env` doesn't touch your login session.

## What you may run unasked

The user's request authorizes the actions it names and the setup those actions
require. **It never authorizes shipping.**

- **Always fine — read-only:** `auth me`, `app config get`, `app share` with no
  argument (status), `app assets push --dry-run`, any `--dry-run`.
- **Authorized by "build me an app":** scaffolding the project files yourself,
  plus `ro app init`. The CMS webapp has to exist before the project can run at
  all — the GameSDK needs a real `WEBAPP_ID` in its iframe — so provisioning is
  part of setting the project up, not part of publishing it. Local work is
  likewise yours: editing files, installing dependencies, running and testing
  the app.
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

## Environments: production, automatically

Creator accounts are production-only, and the CLI resolves every command's default `-e` from
your stored session tier: on a machine whose only sessions are creator-tier, bare commands
(`ro app deploy`, `ro app config push`, `ro app share …`) already target production — no
`-e production` needed. `ro auth me` shows `Tier: creator` when this applies. An explicit `-e`
always wins, `RO_ENV` overrides the default, and adding `-e production` by hand is harmless —
just unnecessary. (The first command in a process prints a one-line stderr notice when the
tier-resolved default isn't `development`: `ℹ️  Using [production] (from your session tier).
Pass -e or set RO_ENV to override.`)

The one remaining landmine is passing `-e development` explicitly: there is no creator
development environment. `ro app init` force-corrects it with an `ℹ️` line; other commands will
fail against the dev host. If a command unexpectedly targets development, check `ro auth me` —
a `Tier: staff` line (or a missing `Tier:` line on a legacy session) means the tier-resolved
default doesn't apply on this machine.

## Getting started, start to finish

```bash
# 1. Log in once per machine (skip if `ro auth me` already shows a session)
ro auth login          # opens the web-client's sign-in + consent page (3 scopes)

# 2. Scaffold your app yourself — any static SPA stack. No cloning, no template.
mkdir wwii-explorer && cd wwii-explorer
#    …create your files (or run /new-ro-app for the guided flow)…

# 3. Provision — writes WEBAPP_ID/DEPLOY_TOKEN to .env, installs the game-sdk docs
ro app init --title "WWII Explorer"

# 4. Build it — your normal dev loop; the build must emit static assets (dist/ preferred)

# 5. Metadata: edit webapp.config.json (title, description, config), then:
ro app config push --dry-run   # preview the delta
ro app config push -y          # apply (drop -y if you want the interactive Y/N prompt)

#    Cover image goes through assets, then into the config file:
ro app assets push ./cover.png --dest images --json   # → public URL
#    put that URL into webapp.config.json's "coverImg", then:
ro app config push -y

# 6. Ship it — live in production, visible only to you until you share it
ro app deploy

# 7. Go live, one rung at a time (or jump straight to public)
ro app share family -y     # instant
ro app share school -y     # needs a teacher/school-admin role at your school
ro app share public -y     # files a review request — staff approve it
```

## Auth: login, me, logout

- **`ro auth login`** — browser PKCE flow, production by default (no `-e` needed). The browser opens the **web-client's own sign-in and consent page** — the same site you already use, not a separate admin tool — at `https://app.rodyssey.ai/auth/cli-authorize`. If you're not already signed into the web-client, you'll see its normal login first and land on the consent screen right after. The consent screen shows exactly three scopes for creator accounts: `webapps:create`, `webapps:deploy-token:create`, `webapps:config`. On success it prints `✅ Logged in to CMS [production]` and the CMS URL. Headless/remote sessions: add `--no-open` to print the URL instead of opening a browser, and relay it to a human to complete in any browser — there is no fully non-interactive login. `--callback-port <p>` pins the local callback port (useful over SSH port-forwarding; default is a random free port). The command itself **blocks and doesn't exit** until the human finishes the consent screen and the local callback lands (default timeout 5 minutes, `--timeout <seconds>` to change it) — so for a headless agent the flow is just: run it with `--no-open`, relay the URL, then wait for the command to exit on its own (no polling needed), and run `ro auth me` afterward as confirmation.
  - **If the account isn't authorized for the CLI at all** (not a `cli-creators` member and not staff), the consent page redirects back before showing any scopes, and the CLI prints `Error: CMS login failed: access_denied: Your account is not enabled for the Rodyssey CLI. Ask an RO admin to add you to the cli-creators group.` That's an eligibility problem, not a CLI bug — ask an RO admin to add the account to the `cli-creators` group, then try again.
  - **Landed on the wrong account's consent screen?** This page has no in-page "switch account" link — if you're already signed into the web-client as someone else, sign out of the web-client entirely (not just close the tab) and re-run `ro auth login`. (Advanced/rare: `--consent-url <url>` or `RO_CONSENT_URL` can point the browser at a different web-client host — not needed for normal use.)
- **`ro auth me`** — prints `Logged in as: <email>`, a `Tier: creator` line (the signal that the tier-resolved production defaults apply — legacy sessions predating the identity-aware flow print no `Tier:` line), `Granted scopes: webapps:create, webapps:deploy-token:create, webapps:config`, and — if your account is tied to a school — `Acting for school: <name> (id: <id>)`. That acting-school line is exactly what `ro app init --school` and `ro app share school` check against. Add `--remote` for a server-side freshness check.
- **`ro auth logout`** (`--all` for every stored environment) — revokes the session server-side (so it can't be reused even if the local file leaks) and forgets it locally. If the server call fails (offline, already revoked), it still forgets locally and prints a warning that server-side revocation didn't happen.

## Starting a new app — scaffold it yourself (no clone)

There is no template download. You (or your agent) write the project files with any stack you
like, then provision from inside the folder:

```bash
ro app init --title "My App"            # your own app
ro app init --title "My App" --school   # school-owned: requires a teacher/school-admin role
```

The platform contract for what you scaffold:

- the build must emit **static browser assets** with at least one `.html` file (`dist/`
  preferred; otherwise pass `--dist-dir <folder>` to `ro app deploy`)
- **no server-only code** in the deployed app
- **GameSDK is injected by the platform at runtime — never bundle it**; use the Rodyssey
  iframe wrapper for local SDK testing (see `.agent/skills/game-sdk/SKILL.md` after init)
- no credentials in source

`ro app init` provisions the CMS webapp (a **private draft** — `VIBED_APP`, share mode
`private`, visible to nobody but you until you `ro app share`), writes
`WEBAPP_ID`/`DEPLOY_TOKEN` to `.env`, and installs the GameSDK docs into
`.agent/skills/game-sdk/`. Add `--with-sdk-files` for `game-sdk.d.ts` typings. It needs a
stored session — without one it fails with a clear `No production session found` error (run
`ro auth login`, then retry). **`--school`** requires you to currently hold a teacher or
school-admin role at your acting school (see `ro auth me`); the server validates and rejects
with `share_rung_forbidden` otherwise. `createdBy` still records you as the individual creator
either way.

`ro app create` is **staff-only** and prints a redirect to this flow if a creator runs it.
For the guided end-to-end version of this section, run `/new-ro-app`.

## `app deploy`

```bash
ro app deploy
```

Auto-detects the SPA build pipeline (build → upload assets in batches → zip + upload HTML → trigger the CMS deploy) and prints the public app URL when it finishes. Non-`dist` output: `--dist-dir <folder>`; custom build: `--build-command "<cmd>"`. After a successful deploy: if `webapp.config.json` differs from the CMS you're prompted to push it (or it auto-pushes with `--push-config`); if the file is simply missing, the CLI pulls it for you automatically, no prompt.

## `webapp.config.json` and `app config`

- **`ro app config pull`** — GETs the CMS config and writes a clean `webapp.config.json` (the `{ webappId, updatedAt }` envelope stripped, empty defaults like `tags: []` removed).
- **`ro app config push`** — reads `webapp.config.json`, previews a New/Update/Delete diff against the CMS, prompts to confirm (`-y` to skip, `--dry-run` to preview only), then PATCHes. Only keys present in the file are touched; clear a field with an explicit `null`.
- **`ro app config set --title ... --description ... --cover-img ...`** — one-off field updates without touching the file.

**What you can write** (allowlisted for creator-tier apps): the branding fields — `title`, `description`, `coverImg`, `localization`, `widgetManifest` — and the free-form `config` runtime blob. Everything else — `tags` (including age-rating tags), `characterIds`, `knowledgeIds`, `imageStyleIds`, `giftBoxConfig`, and the flashcard fields — is rejected with `config_key_forbidden` naming the key. Those stay staff-curated: content-ID fields reference CMS entities you can't enumerate, and age rating is assigned during the `public` review step, not set by hand.

Use **`coverImg`** for the cover image. There is no `coverUrl` field for creator apps — don't invent one; only `coverImg` is ever accepted.

**Check i18n whenever the user approves a push or deploy.** `title`/`description` are the launcher copy families and schools browse; `localization` holds their translations. Before running the approved `config push`/`config set` (or a deploy that will publish metadata), verify localization covers the languages the app's audience uses, show the user the exact strings per language, and confirm the wording with them. Draft translations if asked — but the user signs off; never push machine-translated copy unseen.

⚠️ Never seed `webapp.config.json` from `ro app config get --out` — only `pull` strips the CMS's empty defaults. Feeding a raw `get` response into `push` clears real fields (they read as explicit clear-signals).

## `app assets push` — cover images and other files

```bash
ro app assets push ./cover.png --dest images --json
# → { "success": true, "assets": [{ "path": "images/cover.png", "url": "https://…", "size": 12345 }] }
ro app config set --cover-img "<url>"
# — or put the URL in webapp.config.json's coverImg and `ro app config push`
```

Re-pushing a path **overwrites** it (the public cache is 5 minutes — prefer a new filename over reusing a path that's already referenced, so users don't briefly see the stale cached version). No confirmation prompt (machine-to-machine, like `app deploy`); use `--dry-run` to preview the local→remote mapping.

## The share ladder — `app share`

Your app starts **private**. Going live means climbing four rungs: `private → family → school → public`.

**Rungs are directly addressable, not a required sequence.** `ro app share <rung>` always requests that exact rung — there's no client- or server-side requirement to pass through the ones in between. Going straight from `private` (or `family`) to `public` is completely normal. `school` only matters if you actually want school-level visibility and hold the role for it; if you don't, you simply never use it — jump from `family` straight to `public` instead.

```bash
ro app share                     # print the current rung (+ pending review, if any)
ro app share family -y            # instant if you're your linked family's billing holder
ro app share school -y            # instant IF you hold teacher/school-admin at your acting school
ro app share public -y            # files a review request — not applied yet
ro app share private -y           # downgrade — always allowed
```

Any call that names a rung first prints a banner — `Change visibility of <webappId> to [<rung>] on [production]` — before doing anything else; without `-y`/`--yes` on a TTY it then asks `<same banner>? (y/N):`. On success (any rung except a pending `public` request) it prints `✅ Share mode set to [<rung>].`.

- **`family`** — applies immediately, owner self-serve, **provided you're the primary/billing account holder of an RO family linked to your account**. Makes the app visible to your RO family: the household linked to your account. Family members see it on their RO home. If there's no linked family, or you're not its billing holder, the server rejects with `share_rung_forbidden` and the CLI appends `Family sharing requires an RO family linked to your account (you must be the family's primary/billing account holder).` — see "Typed errors" below.
- **`school`** — applies immediately, but only if you currently hold a teacher or school-admin role at the acting school on your session (`ro auth me` shows this as `Acting for school: ...`). The server re-checks your role at request time, not just at login — if your role changed since you logged in, re-run `ro auth login`.
- **`public`** — never applies immediately for creator-tier callers. The response is a pending review (HTTP 202); the CLI prints ``⏳ Public review requested — status: pending. Check later with `ro app share`.``. A staff reviewer approves (or denies) the request out of band and assigns the age rating — there is nothing further to do from the CLI. Check back any time with `ro app share`: a pending review shows as `⏳ Public review pending (requested <date>)` beneath the current rung.
- **Downgrades** (moving back down the ladder, including all the way to `private`) are always owner self-serve, no role check.
- Running it with **no argument** never changes anything — it only prints the current rung and any pending review.
- **Non-TTY:** without `-y`/`--yes` the command errors out — pass `--yes` or `--dry-run` in scripts/agents.

## Typed errors

The CLI surfaces certain server error codes with extra context. The exact shape differs by which command hit it — verified against source, don't assume they're identical:

| Code | Where you'll see it | What the CLI actually prints |
|---|---|---|
| `not_eligible` | `ro app init` | One line, the server's message verbatim, e.g. `Your account is not enabled for CLI app creation. Ask an RO admin to add you to the cli-creators group.` |
| `share_rung_forbidden` | `ro app init --school` | One line, verbatim, e.g. `School-owned apps require a teacher or school-admin role at your acting school.` |
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
| `app init` | prompt-free by design once a session is stored — no picker, no confirmation. Without a stored session it fails fast with the clear `No production session found` error (never hangs). |
| `app config push` | errors out — pass `--yes` or `--dry-run` |
| `app share <mode>` | errors out — pass `--yes` or `--dry-run` |
| `app deploy` config-drift check | warns only if `webapp.config.json` differs from the CMS (auto-push instead with `--push-config`); a missing file is auto-pulled silently either way |
| `app config set` / `app assets push` | prompt-free by design (deploy-token, machine-to-machine) |

`ro auth login` is never headless — the browser consent screen always needs a human; `--no-open` only skips auto-opening the browser, it still prints the URL for a human to complete elsewhere.

**Pre-flight for any headless/agent run that includes `app init`:** run `ro auth me` first and confirm it shows a session with `Tier: creator`. If it doesn't, log in (`ro auth login`) before calling `init`.

## Common pitfalls

- **`ro app create` says "staff-only"** — expected, not an error in your setup: creators scaffold their own files and provision with `ro app init` (or the guided `/new-ro-app`). Don't retry `create` with different flags.
- **`init` failed with "No production session found"** — no stored session, or your only session was stored under `development` (from an explicit `ro auth login -e development`). Log in with bare `ro auth login` — it lands on production — then retry.
- **A command unexpectedly targeted development** — you passed `-e development` explicitly, or this machine also holds a staff session (`ro auth me` shows `Tier: staff`), which flips the default. See "Environments" above.
- **`config push` wiped fields** — you seeded `webapp.config.json` from `ro app config get --out` instead of `ro app config pull`. Only `pull` strips the CMS's empty defaults (`tags: []`, etc.); pushing those back is treated as a real clear signal. Re-seed with `pull`.
- **`config_key_forbidden` on push/set** — you (or a stale config file) included a staff-curated key — `tags`, `characterIds`, `knowledgeIds`, `imageStyleIds`, `giftBoxConfig`, or a flashcard field. Remove that key; only the branding fields and `config` are writable on creator-tier apps (see "What you can write" above).
- **`ro app release` doesn't work for you** — that's expected; creator-tier apps (`VIBED_APP`) reject it with `apptype_locked`. Use `ro app share` to change visibility instead.
- **Cover image not showing after a re-push** — the public asset cache is 5 minutes; prefer a new filename when replacing an already-referenced image rather than waiting it out.
- **`ro app share public` seems stuck** — it's supposed to: the rung only changes once a staff reviewer approves the request. `ro app share` shows the pending status; there's no CLI action that speeds this up.
- **`ro auth login` shows a consent screen for the wrong person** — you (or whoever's at the keyboard) is already signed into the web-client as someone else. There's no switch-account link on this screen — sign all the way out of the web-client and re-run the login command. See "Auth: login, me, logout" above.
