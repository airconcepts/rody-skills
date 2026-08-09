# rody-skills

Agent skills and slash commands for building on Rodyssey with `@rodyssey/cli` (`ro`).

This repo is the canonical, public distribution home for the **creator-tier** agent
assets. (RO staff use a separate internal edition that does not live here.)

| Asset | What it does |
| --- | --- |
| `skills/rody-creator/SKILL.md` | Teaches an AI agent the creator-tier `ro` surface: scaffold your own app (no template clone), provision it with `ro app init`, deploy it, manage `webapp.config.json`, push assets, and go live via the share ladder (`private → family → school → public`). |
| `skills/webapp-deploy/SKILL.md` | Tier-neutral deploy reference: build requirements, `ro app deploy` variants (`--dist-dir`, `--build-command`, `--url`), and metadata. Also installed project-local by the CLI. |
| `commands/new-ro-app.md` | `/new-ro-app` — guided birth of a new app: art direction → scaffold → `ro app init` → tracking design → local review → stops before deploy. **This is the recommended way to start a new app.** |
| `commands/go-live.md` | `/go-live` — guided ladder climb: pre-flight checks → current rung → target rung → share. |
| `commands/ro-doctor.md` | `/ro-doctor` — diagnoses `ro` auth, environment, token, and scope problems and prints exact fixes. |

The CLI itself is public on npm:

```bash
npm i -g @rodyssey/cli
```

## Install the skill

### Claude Code

```
/plugin marketplace add airconcepts/rody-skills
/plugin install rody-creator@rody-skills
```

The commands are then available as `/rody-creator:new-ro-app`, `/rody-creator:go-live`, and `/rody-creator:ro-doctor`.

### Cursor

```bash
git clone https://github.com/airconcepts/rody-skills
mkdir -p ~/.cursor/skills ~/.cursor/commands
cp -R rody-skills/skills/rody-creator rody-skills/skills/webapp-deploy ~/.cursor/skills/
cp rody-skills/commands/*.md ~/.cursor/commands/
```

### Grok

```bash
grok plugin install https://github.com/airconcepts/rody-skills --trust
```

### Codex and other AGENTS.md-driven agents

Copy `skills/rody-creator/SKILL.md` into your project (for example
`.agent/skills/rody-creator/SKILL.md`) and add this block to your `AGENTS.md`:

```markdown
<!-- BEGIN RODY-CREATOR SKILL -->
When the task involves the Rodyssey CLI (`ro`) — creating, deploying,
configuring, or sharing a webapp — first read
`.agent/skills/rody-creator/SKILL.md` and follow it.
<!-- END RODY-CREATOR SKILL -->
```

## Quickstart — starting a new app

**Recommended: let your agent drive.** In your agent chat, type the command with an app name
*or* just describe the app:

```
/new-ro-app about fractions for grade 4
```

The agent settles the art direction with you, scaffolds the app (no template, no cloning),
provisions it with `ro app init`, wires in learning tracking, then **starts the dev server and
hands you the local URL to review** — it stops there and asks before ever deploying. When you
like what you see, `/go-live` walks the share ladder.

Plain language ("create a ro app about …") also works — the `rody-creator` skill picks it up —
but `/new-ro-app` guarantees the full guided flow.

**Doing it by hand** (no agent), the same flow is:

```bash
ro auth login                  # browser sign-in + consent (production is automatic)
mkdir my-app && cd my-app      # write your own static SPA here — any stack
ro app init --title "My App"   # provisions the webapp, writes .env + webapp.config.json
ro app deploy                  # when you're ready to ship (stays private until shared)
ro app share family -y
```

> `ro app create` is the RO-staff template path — creator accounts are pointed to the flow
> above automatically.

## Updating

Update through the same channel you installed with (marketplace update, re-copy,
or plugin re-install). The plugin version tracks the `@rodyssey/cli` release line
the content was last verified against — see `.claude-plugin/plugin.json`.

---

© Air Concepts. Provided for use with Rodyssey products.
