# AGENTS.md — how this repo relates to the others

This repo is the **canonical, public home** of the creator-tier Rodyssey agent
assets: the `rody-creator` skill and the `/go-live` and `/ro-doctor` commands.
Agents (and humans) editing these assets edit them **here** — other repos hold
mirrors.

| Repo | Visibility | Role |
| --- | --- | --- |
| `rody-skills` (this repo) | public | **Canonical** for `skills/rody-creator/SKILL.md`, `commands/go-live.md`, `commands/ro-doctor.md`. |
| `ro-cli` | private | Source of the `ro` CLI itself, plus the **staff** skill (`rodyssey-cli`) and `/ship-prod`. Holds **mirrors** of this repo's three files, synced via `bun run sync-public-skills` there — never edit the mirrors. |
| `ro-home` | private | The Rodyssey CMS + web client the CLI talks to; server-side creator-tier behavior (eligibility, share ladder, consent) lives there. |

## Edit rules

1. Content changes to the skill or commands happen in **this repo**.
2. Bump the version in `.claude-plugin/plugin.json` and `plugin.json` when the
   change is material to users.
3. Then sync ro-cli's mirrors: in a ro-cli checkout that has this repo as the
   sibling directory `../rody-skills`, run `bun run sync-public-skills` and
   commit both repos.
4. The staff edition is deliberately absent from this repo — **never** copy
   staff content in (promote, global-config, entity groups, GameSDK actions,
   `/ship-prod`, dev/staging URLs).

Full sync contract: see `MAINTAINING.md`.
