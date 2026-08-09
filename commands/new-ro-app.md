---
description: Guided no-clone birth of a creator-tier webapp — scaffold with your own stack, provision with ro app init, design tracking, stop before deploy
argument-hint: [app name]
---

Create a brand-new creator-tier Rodyssey webapp **without cloning anything**: you scaffold the
files yourself, then `ro app init` provisions the CMS webapp. Follow this procedure exactly. If
the rody-creator skill is available, consult it for background; this command is the procedure,
the skill is the reference. `ro app create` is **staff-only** — on a creator account it prints a
redirect to this flow; don't fight it.

## 1. Pre-flight

1. **Session:** run `ro auth me`. Expect `Logged in as: <email>` and `Tier: creator`. If not
   logged in, run `ro auth login` — it opens the web-client's own sign-in + consent page and
   **blocks until the human finishes it**; in a headless session add `--no-open`, relay the
   printed URL to the user, wait for the command to exit, then confirm with `ro auth me`. On
   `access_denied` ("not enabled for the Rodyssey CLI"), stop: the account needs `cli-creators`
   membership from an RO admin — nothing to fix locally.
2. **Tier:** if `ro auth me` prints `Tier: staff`, stop and tell the user this command is for
   creator accounts (staff scaffold with `ro app create`).
3. **Project directory:** if `$ARGUMENTS` names the app, create that folder and work inside it;
   otherwise ask the user for a name. If the working directory already has a `.env` containing
   `WEBAPP_ID`, STOP — that's an already-provisioned project; changes to it go through the
   normal deploy/config lifecycle, not a new birth.

## 2. Scaffold — your own stack

Build the app with whatever stack you're best at (Vite + React, plain HTML/JS, Svelte, …).
The platform contract:

- The build must emit **static browser assets** with at least one `.html` file; `dist/` output
  preferred, otherwise you'll pass `--dist-dir <folder>` at deploy time.
- **No server-only code** in the deployed app.
- **GameSDK is injected by the platform at runtime — never bundle it.** For local SDK testing
  use the Rodyssey iframe wrapper; details live in `.agent/skills/game-sdk/SKILL.md` once
  step 4 installs it.
- **No credentials in source** — `.env` stays gitignored.

## 3. Visual assets — ask first

If you have image-generation ability, **offer** to generate visual assets (backgrounds, sprites,
icons, cover art) to make the app richer. **Ask the user before generating anything** — never as
a silent default. A cover image is applied later via
`ro app assets push <file> --dest images --json` → put the returned URL into
`webapp.config.json`'s `coverImg`.

## 4. Provision

```bash
ro app init --title "<the app's display title>"
```

For a school-owned app (requires a teacher/school-admin role at the acting school shown by
`ro auth me`):

```bash
ro app init --title "<title>" --school
```

Add `--with-sdk-files` if you want `game-sdk.d.ts` typings in `src/types/`. Then verify:

- `.env` now contains `WEBAPP_ID` and `DEPLOY_TOKEN` (presence only — never print values).
- `.agent/skills/game-sdk/` exists — init installs the GameSDK docs; that's your reference for
  every SDK call you write.

If init errors with "No production session found", run `ro auth login` (no `-e`) and retry.

## 5. Design LMS-meaningful tracking

Design tracking that a teacher or LMS report can evaluate later — outcomes, not clicks. Read
`.agent/skills/game-sdk/tracking-service.md` first, then:

- Wrap sessions: `gameSDK.trackingService("SESSION_START")` on load and
  `gameSDK.trackingService("SESSION_END")` on exit.
- Emit learning events with meaningful payloads, e.g.
  `gameSDK.trackingService("QUIZ_ANSWERED", { topic: "fractions", correct: true, attempts: 2 })`.
- Never use the reserved types `APP_DATA`, `PLAYER_DATA`, or `CUSTOM_DATA`.

## 6. Build and run locally

Normal dev loop: install dependencies, run the dev server, verify the app in a browser. SDK
calls only work inside the platform iframe — verify pure-UI behavior locally and SDK behavior
after the user deploys.

## 7. Stop

Do **not** deploy, push config, or share unasked — those are the user's calls (rody-creator's
"What you may run unasked" applies). Report what you built and ask one question: *"It's ready —
want me to deploy it?"* When the user wants others to see it, `/go-live` is the guided ladder
climb.
