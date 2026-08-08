# rody-skills

Agent skills and slash commands for building on Rodyssey with `@rodyssey/cli` (`ro`).

This repo is the canonical, public distribution home for the **creator-tier** agent
assets. (RO staff use a separate internal edition that does not live here.)

| Asset | What it does |
| --- | --- |
| `skills/rody-creator/SKILL.md` | Teaches an AI agent the creator-tier `ro` surface: create your app, deploy it, manage `webapp.config.json`, push assets, and go live via the share ladder (`private → family → school → public`). |
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

The commands are then available as `/rody-creator:go-live` and `/rody-creator:ro-doctor`.

### Cursor

```bash
git clone https://github.com/airconcepts/rody-skills
mkdir -p ~/.cursor/skills ~/.cursor/commands
cp -R rody-skills/skills/rody-creator ~/.cursor/skills/
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

## Quickstart (what the skill teaches)

```bash
ro auth login -e production    # browser sign-in + consent
ro app create my-app           # provisions the webapp, writes .env + webapp.config.json
cd my-app && bun install && bun run dev
ro app deploy -e production    # when you're ready to ship
ro app share family -e production -y
```

## Updating

Update through the same channel you installed with (marketplace update, re-copy,
or plugin re-install). The plugin version tracks the `@rodyssey/cli` release line
the content was last verified against — see `.claude-plugin/plugin.json`.

---

© Air Concepts. Provided for use with Rodyssey products.
