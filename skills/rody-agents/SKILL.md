---
name: rody-agents
description: Use this skill when a teacher or school-admin uses the @rodyssey/cli tool (`ro`) to author LMS 1:1 chat agents for their school — creating and editing agent drafts, managing the school knowledge library, applying AI-proposed suggestions, verifying by chatting with the draft, publishing to students, and importing content packs. Trigger on `ro agent *` (list/get/new/set/refine/accept/questions/say/publish/push), `ro knowledge *` (list/get/images/add/set/rm), or `ro character list`; on requests like "create a chat agent for my class", "build a revision buddy from this deck", "upload this PowerPoint as school knowledge", "make the agent warmer", "apply the suggestions", "generate quiz questions", "publish my agent", "import this content pack"; on questions about objectives, evaluation hints, rubrics, behavior rules, tone knobs, aiGuidance, mediaQuestions, offerOptions, openDrawing, or pack.json; and on LMS errors like scope_missing, acting_school_required, not_lms_user, lms_disabled, school_mismatch, override_locked, knowledge_not_ready, character_not_found, draft_invalid, ai_points_exhausted, ai_failed. This is the teacher tier — for building and shipping your own webapp (scaffold, init, deploy, share ladder) use the `rody-creator` skill; RO staff workflows live in the staff `rodyssey-cli` skill.
---

# Rodyssey CLI — LMS Agent Authoring (Teacher Edition)

`ro agent`, `ro knowledge`, and `ro character` author **LMS 1:1 chat agents** from the CLI — the same objects your school's LMS "Agents" tab edits in the browser. You always edit a **draft**; students only ever see what you **publish**.

**Contents:** Command map · Auth & the school session · The authoring loop · Knowledge handling · Two edit tracks: `set` vs `refine`/`accept` · Objectives, rubrics, wrap-up · Characters: choose, don't forge · Quiz questions · Verify with `say` · Publish · In-chat affordances · Content packs · Typed errors · What you may run unasked · Common pitfalls

## Command map

```
ro agent list                          [--json]
ro agent get <agent-id>                [--json] [--out <file>]
ro agent new  --name <name> [--about <text>] [--character <slug-or-id> | --generic]
              [--knowledge <ids...>] [--language <code>] [--dry-run]
ro agent set <agent-id>                <field flags — see "Two edit tracks"> [--dry-run]
ro agent refine <agent-id> "<intent>"  [--source <knowledge-id>] [--json]
ro agent accept <agent-id> --suggestion <picks> [--dry-run]
ro agent questions <agent-id> --from <source-id | topic:<instruction>> [--count <n>] [--dry-run]
ro agent say <agent-id> "<message>"    [--session <file>]
ro agent publish <agent-id>            [-y|--yes] [--dry-run]
ro agent push <pack-dir>               [-y|--yes] [--character-map slug=existing-slug ...]
                                       [--allow-text-mode] [--dry-run]

ro knowledge list                      [--json]
ro knowledge get <source-id>           [--json]
ro knowledge images <source-id>        [--json]
ro knowledge add <file-or-url>         [--title <t>] [--ai-guidance <text>]
                                       [--language auto|zh-HK|zh-TW|en]
                                       [--wait] [--wait-timeout <seconds>]
ro knowledge set <source-id>           [--title <t>] [--ai-guidance <text>] [--tags <csv>]
ro knowledge rm <source-id>            [-y|--yes]

ro character list                      [--json]
```

Every command above also accepts `--school <id-or-name>` — a guard, not a selector (see next section).

## Auth & the school session

Log in once with `ro auth login` and grant the **`lms:agents`** scope on the consent screen. It is grantable by any teacher or school-admin whose account is linked to a school — no other scope, token, or flag is involved on this surface, and there is no per-project `.env` here.

The first LMS command silently exchanges your CLI session for an LMS session bound to **one school — yours** (cached in `~/.rodyssey/config.json`). Everything you list, edit, or publish belongs to that school.

`--school <id-or-name>` never *selects* a school — it *asserts* the session's school matches and refuses with `school_mismatch` otherwise. Use it in scripts to be sure you're acting where you think you are.

## The authoring loop

See → ingest → author → verify → publish:

```
ro knowledge list && ro character list                      # ground first: what already exists?
ro knowledge add ./deck.pptx --wait                         # binaries fine; --wait polls the pipeline
ro knowledge set <id> --ai-guidance "第二節係俾 agent 扮錯用"     # editorial judgment, not summary
ro agent new --name "植物偵探" --character rody --knowledge <id>
ro agent set <id> --objectives ./objectives.json --max-conversation 8
ro agent refine <id> "warmer, less lecturing, quiz from the deck"
ro agent accept <id> --suggestion about,goals[0],report,tone
ro agent say <id> "我唔知點開始" --session chat.json           # VERIFY before publish
ro agent publish <id>                                       # live audience → confirms
```

## Knowledge handling

`ro knowledge add` takes a local file (PowerPoint, PDF, Word, audio/video — binaries are fine) or a URL. The server pipeline extracts text and images; `--wait` polls until it settles (`ready`/`failed`, default timeout 600s) and `--language` hints transcription (`auto | zh-HK | zh-TW | en`). Agents can only use `ready` sources — attaching one still processing fails with `knowledge_not_ready`; check `ro knowledge get <id>`.

`--ai-guidance` is **editorial judgment, not a summary**: tell the agent what the material is FOR, section by section if needed (e.g. 「第二節係俾 agent 扮錯用 — 唔好照住教」). The pipeline already knows what the document says; only you know how it should be used in class.

`ro knowledge images <id>` lists the images extracted from a source — the raw material for the agent's `--media` entries. `ro knowledge set` edits title / guidance / tags after the fact. `ro knowledge rm` deletes a source (confirmed; detached agents lose that material at their next publish).

## Two edit tracks: `set` vs `refine`/`accept`

**`set` writes fields you already know.** Scalars are flags; longer text takes `<text-or-file>`; structured fields take `<json-or-file>`. The main groups:

- Identity: `--name`, `--about`, `--greeting`, `--persona` (system-prompt body; stays English by convention), `--goals`
- Teaching: `--objectives <json-or-file>`, `--max-conversation <n>`, `--rubrics`, `--insight-guidance`
- Voice: `--tone warmth=playful,verbosity=concise`, `--add-behavior-rule <rule...>`, `--remove-behavior-rule <rule...>`, `--behavior-rules <json-or-file>`
- Cast & material: `--character <slug-or-id>` / `--generic`, `--attach-knowledge <ids...>`, `--detach-knowledge <ids...>`, `--knowledge-mode summary|full|auto`
- Student experience: `--questions <json-or-file>`, `--media <json-or-file>`, `--target-audience`, `--language`, `--language-complexity`, `--chat-type educational|general_chat`, `--chat-mode text|video`, `--allow-camera`/`--no-allow-camera`, `--allow-drawing`/`--no-allow-drawing`, `--suggested-replies`/`--no-suggested-replies`, `--tags <csv>`, `--cover <url>`, `--bg-image <url>`
- Escape hatch: `ro agent get <id> --out draft.json` → edit → `ro agent set <id> --draft draft.json` (replaces the WHOLE draft)

**`refine` asks the AI to propose wording from your intent.** Every suggestion comes back with a reason, and **nothing applies until you `accept`** — the latest proposal is cached per agent. Pick what to apply:

```
ro agent accept <id> --suggestion about,goals[0],goals[*],insight,rubrics,report,tone
```

Picks: `about` · `goals[N]` (one goal) · `goals[*]` (all) · `insight` · `rubrics` · `report` · `tone` — comma-separated, any subset. `--source <knowledge-id>` grounds a refine in one specific source. `refine` never edits the draft; `accept` on stale picks errors (e.g. "No goal suggestion at index 2") rather than guessing.

## Objectives, rubrics, wrap-up

Objectives are `[{description, evaluationHint?}, …]` — `--objectives` replaces the whole set. The best `evaluationHint`s say what does **not** count: 「只重複 agent 講過嘅嘢唔算達成」beats 'student understands photosynthesis'. Rubrics (`--rubrics`) and report guidance (`--insight-guidance`) shape the post-chat report teachers receive.

`--max-conversation N` wraps the session up after N exchanges by managing a magic `__max-conversation__` objective for you (0 clears it). **Never hand-edit that entry** inside an `--objectives` array or a `--draft` round-trip — set the flag instead.

## Characters: choose, don't forge

Agents wear an existing school character or none (`--generic`). Characters need a face — an avatar and preview image — which only the proper creation path produces, so the CLI **never creates characters**: run `ro character list` and pick. An unknown slug falls back to the school's `rody` character (interactive runs offer the roster first); create new characters in the LMS **Characters** tab, then re-run.

## Quiz questions

`ro agent questions <id> --from <source-id>` generates questions grounded in a knowledge source; `--from topic:<instruction>` generates from a topic instead; `--count <n>` sizes the batch. Generated questions land in the draft's question bank (replace wholesale with `set --questions`). Keep the bank small and specific — it feeds the in-chat `offerOptions` affordance below.

## Verify with `say` — the point of the loop

Behavior rules, tone, and media anchors **cannot be verified by reading the draft back** — talk to the draft as a student:

```
ro agent say <id> "我唔知點開始" --session chat.json
ro agent say <id> "點解會咁？"   --session chat.json     # same file = same conversation
```

`--session <file>` keeps the conversation across calls. `say` spends the school's AI budget: `ai_points_exhausted` means top up (ask your school admin), `ai_failed` means the model call failed — retry.

## Publish

`ro agent publish <id>` copies the draft live and creates/updates the backing 1:1 chat webapp. Know before you run it:

- **A re-publish of an agent students already have pushes to them immediately** — hence the confirmation (`-y` skips it in scripts, never as a substitute for the user's go-ahead).
- Draft edits change nothing for students until the next publish.
- The chat always runs the **agentic** engine; **video** is the platform-default chat mode — keeping `--chat-mode text` requires an interactive confirmation or `--allow-text-mode`.
- An empty `bgImageUrl` is seeded with the school's default stage background at publish.
- **The opening must orient**: one short line of who am I, what we'll do together, and how to answer — before the first question.
- **Phantom-visual warning**: a greeting that "shows" something (呢個係我以前個樣 / 你睇呢張圖) with empty media leaves students staring at nothing. The CLI warns; the real check is `ro agent say`.

## In-chat affordances (agentic engine)

- **`<media id/>` anchors** — the agent reveals an attached image/video mid-chat. Every `--media` entry needs a `description` (it feeds the model's media database — the CLI warns on missing ones; a description-less entry is dead in-chat).
- **`offerOptions`** — presents a tappable multiple-choice question. Author a small question bank and invite it: for lower forms add a behavior rule like 「二選一用 offerOptions 俾學生㩒，唔好要佢打字」.
- **`openDrawing`** — on drawing-enabled agents (`--allow-drawing`), opens the student's drawing pad with a prompt; the submitted drawing returns to the model as an image. Author 「call openDrawing…」 steps — never ask students to draw somewhere the agent can't see.

## Content packs — `ro agent push <dir>`

A pack is a directory: `pack.json` (`{"name": "..."}`) + `knowledge/*.md` (frontmatter `title:` / `aiGuidance:`) + `scenes/*.json` (full scene drafts; `characterId`/`knowledgeIds` may hold slugs — uuid-shaped values pass through) + optional `assets/`. Scene media may use pack-relative urls like `assets/hanzi/shan.svg` — push uploads each unique file into the school's storage and rewrites the urls, so packs stay portable (absolute and data urls pass through untouched). All-local validation runs before any write; `--dry-run` plans without writing.

Identity and updates: scene ids derive from the pack name + scene slug + your school, so **re-pushing updates the same agents in place — renaming the pack or a scene slug orphans the live agent and creates a new one**. Knowledge upserts by TITLE within the school. Unknown character slugs follow choose-don't-forge above (`--character-map ah-zung=<existing-slug>` overrides per slug).

## Typed errors → fixes

| Error | Fix |
|---|---|
| `scope_missing` / `unauthorized` | Re-run `ro auth login` and grant `lms:agents` on the consent screen |
| `acting_school_required` | Your login isn't bound to a school — re-login; if your account has no linked school, ask your school admin |
| `not_lms_user` | Your account needs the teacher or school-admin role — ask your school admin |
| `lms_disabled` | The school's LMS flag is off — ask RO |
| `school_mismatch` | Your `--school` guard disagrees with the session's school — check `ro auth me`, drop or fix the flag |
| `override_locked` | This is a centrally managed agent; publish is refused by design |
| `knowledge_not_ready` | Source still processing or failed — `ro knowledge get <id>`; re-add on `failed` |
| `character_not_found` | `ro character list` and pick an existing slug |
| `draft_invalid` | A field failed validation — the error names the field path; fix and re-run |
| `ai_points_exhausted` | School AI budget empty — top up, then retry |
| `ai_failed` | Model call failed — retry; if persistent, ask RO |

## What you may run unasked

The user's request authorizes the actions it names and the setup those actions require. **It never authorizes publishing to students or deleting school data.**

- **Always fine — read-only:** `agent list/get`, `knowledge list/get/images`, `character list`, `auth me`, any `--dry-run`.
- **Authorized by "build me an agent":** the authoring loop is the work — `knowledge add`/`set` for materials the user gave you, `agent new`, `set`, `refine`, `accept`, `questions`. A few `agent say` verification turns are part of authoring (the skill mandates verification) — but they spend the school's AI budget, so say you're doing it and stop at `ai_points_exhausted`.
- **Stop and ask, every time:** `agent publish` (live students — a re-publish ships instantly to assigned classes), `agent push` without `--dry-run` (same reach), `knowledge rm` (destroys school material). Build the draft, verify it, then ask one question: *"The draft holds up in chat — want me to publish it?"*

**`-y`/`--yes` suppresses the CLI's own confirmation prompt. It is not the user's authorization.** It exists so an action the user already asked for doesn't hang in a non-TTY. Never reach for it to skip asking.

| Rationalization | Reality |
|---|---|
| "Publish is how I check it works" | `ro agent say` is how you check it works. Publish is how students receive it. |
| "It's just a draft tweak — republish to keep things in sync" | If the agent is assigned, republish pushes the tweak to students immediately. |
| "The old knowledge source is superseded — I'll clean it up" | Other agents may attach it. Show `knowledge list` and ask. |

## Common pitfalls

- **"I refined but nothing changed"** — `refine` only proposes. Apply with `accept --suggestion …`.
- **"My edits aren't showing for students"** — students see the published copy; drafts change nothing until the next `publish`.
- **`knowledge_not_ready` right after `add`** — the pipeline is asynchronous; use `--wait` or poll `ro knowledge get`.
- **Wrap-up fired at the wrong time** — someone hand-edited `__max-conversation__`. Clear with `--max-conversation 0`, then set the intended N.
- **Re-pushed a pack and got duplicate agents** — the pack or scene slug was renamed; identity derives from those names. Keep them stable.
- **Greeting promises a picture nobody sees** — media entry missing or description empty. Fix `--media`, then verify with `say`.
