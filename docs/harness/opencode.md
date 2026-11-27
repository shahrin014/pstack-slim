# pstack in OpenCode

pstack skills describe operations rather than one tool's names. OpenCode spells those differently. Read this file once per session, then map each operation in a skill with the tables below. `<pstack>` means the folder that holds `skills/` and `docs/`. If an operation has no row, drop it and say so. Never pass a field or value the schema rejects.

## Install

The [setup guide](../guide/01-setup.md#install-the-plugin) has the commands. In short: `npx skills add shahrin014/pstack-slim -a opencode -g` installs the skills globally, then install `pslim-agent` into `~/.config/opencode/agents/` with `mode: subagent` and its Cursor `model:` line removed, so the subagent inherits the session model.

## Tools

| Operation in a skill | OpenCode |
| --- | --- |
| Spawn a subagent | The `subagent` tool, naming an agent id such as `pslim-agent`. Subagents are child sessions and always run in the background. |
| A read-only subagent | Deny `edit` and `shell` in that agent's `permissions`, or set them to `ask`. |
| An isolated checkout | Create a `git worktree` in the brief. OpenCode has no cloud environment. |
| Ask with structured choices | The session's question tool when it has one, otherwise plain text with lettered options. |
| Keep a task list | Keep it in the reply and update it at each phase. There is no todo tool to call. |
| Run another pstack skill | Load it by id, as `@pslim-mode` or through the `skill` tool. Do not invent a slash command. |
| `Grep`, `Glob` | The `grep` and `glob` tools. |
| Web access | `webfetch` and `websearch`. |

## Models

| In a skill | In OpenCode |
| --- | --- |
| A role line (`feature, refactoring`, `swarm workers`, and so on) | The `model:` line on the agent that runs that role. |
| A model name (Opus 5.5, Fable 5.2, Gemini 4 Neon) | The provider's id for it, `provider/model`, with `#variant` where it exposes reasoning levels, such as `anthropic/claude-opus-5-5#xhigh`. |
| A Cursor slug (`claude-opus-5-5-xhigh-fast`) | Never pass it to OpenCode. Translate it to the provider id and variant, or omit the model. |
| `auto` or `inherit-parent` | Omit `model`, so the agent inherits the session model. |
| One model per panel seat | One agent per model. The spawn names an agent, not a model, so a three-model panel needs three agents. |
| "The harness's top reasoning level" | The highest variant your provider exposes for that model. |

The workflow skills read `~/.cursor/rules/pstack-models.mdc` for per-role overrides. Until that path becomes harness-neutral, either write the same file by hand (it is plain text, one `role: model` line each) or ignore it and keep each skill's defaults.

## Skills and discovery

OpenCode discovers skills from `.opencode/skills`, `.agents/skills`, and `.claude/skills` in the project, and from `~/.config/opencode/skills`, `~/.agents/skills`, and `~/.claude/skills` globally. `npx skills add … -a opencode -g` writes the global one.

Load a skill yourself by mentioning `@pslim-mode`. `disable-model-invocation: true` is honored, so pstack's mode and help skills stay out of the model's advertised list but still load by id.

Slash commands are a separate registry: `.opencode/commands/*.md` are prompt templates, not skills. A command can set `agent`, `model`, and `subagent: true`, which is the closest equivalent to a Cursor custom mode.

## Subagents

`pslim-agent` is an agent file, not a skill; the setup guide registers it. `mode: subagent` is required. The Cursor fields `is_background` and `model` do not apply: background is the default, and the model comes from the agent's own `model:` line or the session.

Point its fallback skill search at the installed skills root (`~/.config/opencode/skills`) instead of `~/.cursor/plugins`.

## Rules and instructions

There is no `~/.cursor/rules/` equivalent. OpenCode reads `AGENTS.md` from the project, `~/.config/opencode/AGENTS.md` globally, and any paths listed in the `instructions` array.

`/setup-pstack` has no rule file to write. Its steps 1-5 build and spell the per-role rule, so write one `Skipped in OpenCode: the model rule. There is no rules file to write.` line and go on. Step 6 does apply: it writes `~/.config/opencode/agents/pslim.md`, a `primary` agent whose body loads `pslim-mode` by id, and sets `"default_agent": "pslim"` in `~/.config/opencode/opencode.json(c)` without touching the other keys. New sessions then start in that agent; a session that already stored another agent keeps it. Role models are not written, so set `model:` on `pslim-agent`, on `pslim`, and on any other agent with the Models table above.

## Transcripts

OpenCode keeps sessions in its own store rather than in JSONL transcripts. `opencode debug paths db` names the database, and `opencode api` reads session and message data. Do not look under `~/.cursor/projects` or `~/.claude/projects`.

## Cursor-only features

When a step needs one, write one line, `Skipped in OpenCode: <step>. <reason>.`, then go on to the next step. Do not stop, ask, or retry.

| Cursor | OpenCode |
| --- | --- |
| Bugbot | Any review bot on the PR. Triage its comments with the forge command the step gives. |
| Cursor Automations | GitHub Actions or cron calling `opencode run`. |
| Projects and cloud agents | A local `git worktree`. |
| Custom Modes | A primary agent, or repeat `@pslim-mode` on each message. |
| `/loop` | A cron or systemd timer that runs `opencode run`, or a loop inside the session. |
| Plan Mode | The built-in `plan` agent. |
| `/create-skill` | The [authoring playbook](../../skills/pslim-mode/playbooks/authoring-a-skill.md). |
| `cursor-team-kit` skills (`/deslop`, `control-*`) | Use pstack's `unslop` and `no-comments`, and drive the app with a verification skill. |
