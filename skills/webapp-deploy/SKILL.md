---
name: webapp-deploy
description: Use when adapting, building, or deploying an existing SPA/simple frontend project to the Rodyssey webapp platform with @rodyssey/cli. Covers preparing env credentials, GameSDK docs, build output requirements, deploy commands, and metadata config.
---

# Rodyssey Webapp Deploy

Use this skill for browser-only SPA/simple frontend projects, including projects that were not scaffolded by a Rodyssey tool.

## Agent Bootstrap Prompt

Give another coding agent this prompt when handing them an existing frontend project:

```text
You are working in an existing SPA/simple frontend project targeting the Rodyssey webapp platform.

First install Rodyssey agent skills:
  bunx @rodyssey/cli@latest app install-skills
If Bun is unavailable, use:
  npx @rodyssey/cli@latest app install-skills

Then read:
  .agent/skills/webapp-deploy/SKILL.md
  .agent/skills/game-sdk/SKILL.md

If the project is not provisioned yet, run:
  bunx @rodyssey/cli@latest auth login
  bunx @rodyssey/cli@latest app init --title "<app title>"

Build the app as a static frontend, keep the output in dist/ or deploy with --dist-dir <folder>, then deploy with:
  bunx @rodyssey/cli@latest app deploy
```

## Setup Workflow

1. Install local skills:

```bash
bunx @rodyssey/cli@latest app install-skills
# or
npx @rodyssey/cli@latest app install-skills
```

This downloads the GameSDK docs into `.agent/skills/game-sdk` and refreshes the project-local Rodyssey skills.

2. Provision the project if `.env` does not contain `WEBAPP_ID` and `DEPLOY_TOKEN`:

```bash
bunx @rodyssey/cli@latest auth login
bunx @rodyssey/cli@latest app init --title "My Webapp"
```

`ro app init` provisions the CMS webapp against your session's environment (a creator session
always resolves to production), writes `WEBAPP_ID` and `DEPLOY_TOKEN` to `.env`, and installs
the local skills. An explicit `-e <env>` always wins if you need to pin an environment. RO
staff: environment selection and staff auth flows are covered by your internal skill.

3. Confirm the project has a build script. The deploy command expects static HTML/JS/CSS output. Default output is `dist/`.

4. Deploy:

```bash
bunx @rodyssey/cli@latest app deploy
```

For non-`dist` projects:

```bash
bunx @rodyssey/cli@latest app deploy --dist-dir build
```

For custom build commands:

```bash
bunx @rodyssey/cli@latest app deploy --build-command "npm run build" --dist-dir build
```

If the project is already deployed on another hosting platform, register that
runtime URL without building or uploading:

```bash
bunx @rodyssey/cli@latest app deploy --url https://already-hosted.example.com
```

## Build Requirements

- Produce static browser assets with at least one `.html` file.
- Prefer `dist/` output when possible; otherwise pass `--dist-dir`.
- Keep server-only code out of the deployed SPA unless using Rodyssey Dynamic Worker scripts.
- If using GameSDK APIs, read `.agent/skills/game-sdk/SKILL.md` first.
- The hosted platform injects GameSDK for deployed HTML. For local development, use the Rodyssey iframe wrapper instead of direct localhost when testing SDK behavior.

## Metadata

After deploy, set launcher metadata:

```bash
bunx @rodyssey/cli@latest app config set \
  --title "My Webapp" \
  --description "Short user-facing description"
```

Use `--localization <json-or-file>` for translated titles/descriptions and `--cover-img <url>` for cover art.
