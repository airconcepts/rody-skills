---
description: Guided no-clone birth of a creator-tier webapp — scaffold with your own stack, provision with ro app init, design tracking, stop before deploy
argument-hint: [app name, or what the app is about]
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
3. **Project directory:** `$ARGUMENTS` may be an app name (`fraction-quest`) or a description
   (`about fractions for grade 4`). From a description, derive a short kebab-case folder name
   and a display title, confirm both with the user in one breath (fold it into the step-2 art
   question — don't ask twice), and keep the description as your brief for what to build. With
   no `$ARGUMENTS`, ask what the app is about. Create the folder and work inside it. If the
   working directory already has a `.env` containing `WEBAPP_ID`, STOP — that's an
   already-provisioned project; changes to it go through the normal deploy/config lifecycle,
   not a new birth.

## 2. Art direction — settle it BEFORE you scaffold

If you have image-generation ability, ask the user **now, before writing any code** — one
question: *"Want me to generate visual assets for this app (backgrounds, sprites, icons, cover
art)? I can also keep it a clean CSS-only look."* **Never generate without a yes** — but never
silently skip the question either: deferring it is exactly how apps end up with empty slots.

Whatever the answer, the contract for step 3 is **no empty visual slots**:

- **Yes** → plan the asset list with the user, generate during scaffolding, and wire every
  asset in. The cover image is applied after provisioning via
  `ro app assets push <file> --dest images --json` → put the returned URL into
  `webapp.config.json`'s `coverImg`.
- **No, or you can't generate images** → design a deliberate no-image look: CSS gradients,
  solid palettes, typography. A monogram or placeholder is acceptable only as an intentional
  design choice — never as a stub awaiting art that hasn't been discussed.

| Rationalization | Reality |
|---|---|
| "I'll offer images later, once it works" | Later never comes — the build shapes itself around the empty slots. Ask before scaffolding. |
| "Placeholders are fine for v1" | Only *designed* placeholders are. Empty boxes and stub monograms pending an offer are not. |
| "Asking now interrupts the flow" | It's one question, and its answer changes what you scaffold. |

## 3. Scaffold — your own stack

Build the app with whatever stack you're best at (Vite + React, plain HTML/JS, Svelte, …),
honoring the art direction settled in step 2. The platform contract:

- The build must emit **static browser assets** with at least one `.html` file; `dist/` output
  preferred, otherwise you'll pass `--dist-dir <folder>` at deploy time.
- **No server-only code** in the deployed app.
- **GameSDK is injected by the platform at runtime — never bundle it.** For local SDK testing
  use the Rodyssey iframe wrapper; details live in `.agent/skills/game-sdk/SKILL.md` once
  step 4 installs it.
- **No credentials in source** — `.env` stays gitignored.

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

Normal dev loop: install dependencies, run the dev server, verify the app in a browser —
including that **every visual slot renders something intentional** (step 2's contract: generated
art or a designed no-image look, never an empty box). SDK calls only work inside the platform
iframe — verify pure-UI behavior locally and SDK behavior after the user deploys.

## 7. Stop

Do **not** deploy, push config, or share unasked — those are the user's calls (rody-creator's
"What you may run unasked" applies). Report what you built and ask one question: *"It's ready —
want me to deploy it?"* When the user wants others to see it, `/go-live` is the guided ladder
climb.
