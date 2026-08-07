---
description: Diagnose ro CLI auth, environment, token, and scope problems and print exact fixes
argument-hint: [environment: development | staging | production | local]
---

Diagnose why the Rodyssey CLI (`ro`) is failing for this project and report exact fixes. Target environment: $ARGUMENTS (default: `development`). If a Rodyssey CLI skill is available (`rody-creator` for creators, `rodyssey-cli` for staff), consult it for background. NEVER print the value of `DEPLOY_TOKEN` or any token — report presence/absence only.

## Collect

1. **Session:** `ro auth me -e <env>` (add `--remote` for a server-side freshness check). Record: logged in or not, identity type (user / service-account / legacy session), and the granted scopes list.
2. **Project env files:** check `.env` and `.env.production` for the presence of `WEBAPP_ID` and `DEPLOY_TOKEN` (presence only — do not echo values). Remember: `.env` holds the development deploy token, `.env.production` the production one; they are not interchangeable.
3. **The failing command**, if the user named one: prefer re-running it with `--dry-run` where supported; use `RO_DEBUG=1` to get a full stack/handshake trace when the error is unclear.

## Map symptoms to fixes

- `401` / "Login expired" → `ro auth login -e <env>`. Opens a browser to the web-client's consent front door by default (staff can jump to the CMS console from there, or via `--login-url <cms-url>/auth/cli-login`); headless: add `--no-open` and relay the URL to the user.
- `403 forbidden: <token kind> does not have the required scope` → the persisted session predates the scope or the consent screen didn't grant it. Re-run `ro auth login -e <env>` and confirm the needed scope: `cms:global-config:read|write` for `global-config`, `cms:entity-groups:read|write` for `group`, `cms:users:token` (+ `cms:users:read` when using `--as <email>`) for `app action`.
- `access_denied` after the consent screen → account gate, and which gate depends on the population: staff accounts must be `superuser`/`admin` AND their school must have `isRoSchool=true`; everyone else (creator-tier) needs `cli-creators` group membership — ask an RO admin to add the account. This is also the login-time failure for an ineligible creator-tier account, not just a staff-account problem. CMS-side fix either way — nothing to change locally.
- Landed on the wrong account's consent screen → depends which front door: on the **CMS console** (`<cms-url>/auth/cli-login`), use its **"Not you? Sign in as a different account"** link (logs out of the CMS session and returns to the same consent URL); note the documented gap that it doesn't clear an underlying ROID SSO session, so ROID's own authorize screen may still default to the wrong account (it has its own "Use Another Account" button as a fallback). On the **web-client** front door (the default `ro auth login` destination), there's no such link — sign out of the web-client entirely, then re-run `ro auth login`.
- `403 FORBIDDEN_WEBAPP_TOKEN_SCOPE` → the `DEPLOY_TOKEN` doesn't match the `WEBAPP_ID` in the request, or a dev token was used against production (or vice versa). Check which env file the command resolved.
- `404 Webapp not found` on `ro app action` → the runtime-token mint used an org that doesn't own the webapp's school. Pass the webapp's `--org`.
- "Refusing to … in non-interactive mode" → the command wants confirmation and there's no TTY. Add `--yes` (or `-y` for `group`), or `--dry-run` to preview.
- Command appears to hang in a script → an interactive prompt is waiting on stdin (e.g. `app promote`'s source-pull prompt, or `app create`'s template picker). Add `--yes` / `--template <t>` respectively.
- `ro auth me` shows `service account: X` unexpectedly → the last login authorized the CLI as service account X on the CMS console's identity picker (the web-client front door has no such picker). Reach that screen again (`--login-url <cms-url>/auth/cli-login`, or "Open staff console" on the web-client page) and pick "Myself".
- Deploy-token commands work but `global-config` / `group` / `create --auto` / `action` fail → different credentials: those need a CLI session token; a deploy token can never call them.
- `not_eligible` on `ro app create` → the session doesn't hold the `webapps:create` scope: either the account isn't a `cli-creators` member (ask an RO admin to add it) or the session predates the grant. Re-run `ro auth login -e <env>` and confirm the scope is checked on the consent screen.
- `share_rung_forbidden` on `ro app share school` or `ro app create --school` → the account doesn't currently hold a teacher/school-admin role at its acting school. Check acting school via `ro auth me -e <env>` (look for the `Acting for school: ...` line); if it's missing, wrong, or the role changed, `ro auth logout -e <env>` and log back in to re-pick.
- `apptype_locked` on `ro app release` → the webapp's `appType` is `VIBED_APP` (creator-tier) — its visibility is managed by the share ladder, not by changing app type. Use `ro app share` instead.
- `config_key_forbidden` on `ro app config set` / `ro app config push` → the key named in the error isn't writable on creator-tier (`VIBED_APP`) apps (content-ID/tag/flashcard fields are staff-curated, not settable there). Drop that key from the payload or `webapp.config.json` and retry.

## Report

Summarize for the user: identity + scopes per environment checked, env-file status (which tokens are present), the diagnosed root cause, and the exact command(s) to fix it. If the fix needs the browser consent screen, say so explicitly and include the login command (with `--no-open` when the session is headless).
