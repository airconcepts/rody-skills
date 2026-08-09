---
description: Guided ladder climb for a creator-tier webapp — pre-flight checks, current rung, and a share to the next rung
argument-hint: [target rung: family | school | public]
---

Take the current webapp project live with the Rodyssey CLI (`ro`), climbing the visibility ladder (`private → family → school → public`). Follow this procedure exactly. If the rody-creator skill is available, consult it for background; this command is the procedure, the skill is the reference. **Production is the only environment for creator-tier apps.** On a creator machine bare commands already resolve to production (see `Tier: creator` in `ro auth me`); the explicit `-e production` below is kept because it's harmless and pins the right environment even when a staff machine is helping out.

## 1. Pre-flight (read-only, run all before sharing)

1. **Session + identity:** `ro auth me -e production`. If not logged in, run `ro auth login -e production` — this opens a browser to the web-client's own sign-in-and-consent page (the same site the user already uses, not a separate admin tool); in a headless session add `--no-open` and relay the printed URL to the user to open themselves. The command **blocks until the human finishes the consent screen** — just wait for it to exit, then confirm with `ro auth me -e production`. Note whether the identity line shows `Acting for school: <name> (id: <id>)` — you'll need to know this for the `school` rung. If the consent screen shows the wrong account, there's no switch-account link on this page — the user needs to sign out of the web-client entirely and re-run the login command.
2. **Provisioned project?** `.env` must contain `WEBAPP_ID` and `DEPLOY_TOKEN`. If missing, this project was never provisioned — run `ro app init --title "<title>"` in the project root (or the guided `/new-ro-app` flow), then re-run this command.
3. **Config in sync:** if `webapp.config.json` exists, run `ro app config push -e production --dry-run` and show the user any pending New/Update/Delete delta. Resolve it (`ro app config push -e production -y`, or edit the file and re-check) BEFORE sharing, so the live app reflects the intended config.
4. **Fresh deploy:** run `ro app deploy -e production` so the rung you're about to open shows the latest build, not a stale one.

## 2. Current rung

Run `ro app share -e production` (no argument). Report the current rung to the user. If it prints `⏳ Public review pending (...)`, stop here and tell the user their public request is still awaiting staff review — there is nothing else to do until a reviewer acts on it.

## 3. Pick the target rung

If `$ARGUMENTS` names a rung (`family`, `school`, or `public`), use it. Otherwise ask the user which one they want. Rungs are directly addressable, not a required sequence: going straight from the current rung to `public` (skipping `school` entirely) is completely normal and does not require ever having held `school`. Only offer `school` as a step worth taking if the user actually wants school-level visibility. Before offering `school`, remind the user it requires them (or, for a school-owned app, any qualifying teacher/school-admin) to hold a teacher or school-admin role at the acting school noted in step 1 — the server re-checks this at request time and will reject with `share_rung_forbidden` if that role isn't held anymore, even if it was held at login.

## 4. Share

```bash
ro app share <rung> -e production -y
```

- **`family`** — applies instantly, owner self-serve, **provided the user is the primary/billing account holder of an RO family linked to their account**. On success, report and stop. On `share_rung_forbidden`, report the CLI's exact remediation line (`Family sharing requires an RO family linked to your account (you must be the family's primary/billing account holder).`) and stop — this isn't fixable from the CLI; the user needs to create or join a linked RO family (and be its billing holder) before retrying. Do not loop or retry the share call yourself.
- **`school`** — applies instantly if the user still qualifies. On `share_rung_forbidden`, report the CLI's exact remediation line (`Raising to school visibility requires a teacher or school-admin role at your acting school (see ro auth me).`) and stop — this isn't fixable from the CLI; the user needs the role granted first.
- **`public`** — does **not** apply immediately. A 202 response means the CLI printed ``⏳ Public review requested — status: pending. Check later with `ro app share`.`` — this **files a review request**, it does not make the app public. Tell the user plainly: a staff reviewer must approve the request (and assign an age rating) before the app is publicly visible, and there is no further CLI action to take. If a review is already pending, the CLI instead reports `share_review_pending` ("A public review is already pending.") — treat that the same as step 2's pending case.

## 5. Report

Tell the user: the resulting rung (or the pending-review status for `public`), the public app URL from the last deploy, and — for `public` specifically — exactly how to check back later: `ro app share -e production`.
