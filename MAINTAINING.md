# Maintaining rody-skills

Canonical files (edit these, here):

- `skills/rody-creator/SKILL.md` — the creator-tier skill
- `skills/webapp-deploy/SKILL.md` — the tier-neutral existing-project deploy
  skill (moved here with the no-clone change; ro-cli keeps a synced mirror at
  the same path for npm packaging — its bootstrap-prompt section heading and
  ```text fence are load-bearing, parsed by ro-cli's `install-skills` at runtime)
- `commands/new-ro-app.md`, `commands/go-live.md`, `commands/ro-doctor.md` — the slash commands

## After any content change

1. **Version bump** (when material to users): `version` in both
   `.claude-plugin/plugin.json` and root `plugin.json` (kept identical; tracks
   the `@rodyssey/cli` release line the content was last verified against).
2. **Sync ro-cli's mirrors**: in a ro-cli checkout with this repo at
   `../rody-skills`, run `bun run sync-public-skills`, then commit ro-cli.
   The mirror list lives in ro-cli's `scripts/sync-public-skills.ts`; ro-cli's
   `scripts/sync-public-skills.test.ts` has a guard test that fails if mirrors
   drift from this repo — but it only runs when someone runs `bun test` inside
   a ro-cli checkout with this repo present as the sibling `../rody-skills`.
   There is no CI enforcement of this; drift can go unnoticed until someone
   happens to run that test locally with both repos checked out side by side.
3. **Probe discipline**: changes to behavioral sections (especially the
   "What you may run unasked" authorization rules) follow the writing-skills
   RED/GREEN probe discipline used in ro-cli — test the wording on fresh agents
   before shipping it.

## Boundaries

- No staff content, ever (promote, global-config, entity groups, GameSDK
  actions, `/ship-prod`, dev/staging URLs). The staff edition lives in the
  private ro-cli repo.
- No LICENSE file and no `license` manifest field — distribution terms are the
  README's © line, by design.

## Update channels (why ro-cli's `upgrade-template` is not involved)

- **Creators** install/update from this repo (Claude plugin marketplace, Cursor
  copy, Grok plugin) — public HTTPS, no ro-cli access.
- **Staff** update their internal edition via `ro app upgrade-template`, which
  fetches from the private ro-cli repo over SSH and is intentionally unchanged
  by the repo split.
