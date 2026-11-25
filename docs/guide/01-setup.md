# Set up pstack

In this page you install the plugin, pick which models pstack uses, and run your first task. Setup is one command plus a short conversation.

## Install the plugin

In a Cursor chat, run:

```text
/add-plugin pstack
```

Cursor confirms the plugin is installed.

In Claude Code, run `/plugin marketplace add cursor/plugins`, then `/plugin install pstack@cursor-plugins`. Claude Code prefixes plugin commands, so each command in this guide takes a `pstack:` prefix there, such as `/pstack:setup-pstack`, which writes its model rule to `~/.claude/rules/pstack-models.md`. The [Claude Code adapter](../harness/claude-code.md) covers the other differences.

In OpenCode, install the skills with the [skills cli](https://skills.sh/docs/cli) (Node 22.20 or later). `-g` makes them available to every project; drop it to install into the current project's `.agents/skills/` instead:

```bash
npx skills add shahrin014/pstack-slim -a opencode -g --skill '*' -y
```

The CLI installs skills only. `pslim-mode` also delegates to a `pslim-agent` subagent, so install that too. OpenCode needs `mode: subagent`, and the Cursor model slug in the file is not valid there, so both get fixed on the way in:

```bash
mkdir -p ~/.config/opencode/agents
curl -fsSL https://raw.githubusercontent.com/shahrin014/pstack-slim/main/agents/pslim-agent.md \
  | sed -e 's/^is_background: true$/mode: subagent/' \
        -e '/^model: /d' \
        -e 's|~/.cursor/plugins/|~/.config/opencode/skills/|g' \
  > ~/.config/opencode/agents/pslim-agent.md
```

OpenCode loads a skill by id rather than a slash command, so ask for `@pslim-mode` where Cursor takes `/pslim-mode`. Later, `npx skills update` moves the installed skills to newer commits. The [OpenCode adapter](../harness/opencode.md) covers the other differences.

## Pick your models

Run:

```text
/setup-pstack
```

[`/setup-pstack`](../../skills/setup-pstack/SKILL.md) detects the models you have access to, asks for a reasoning budget, shows you each role (code delegates, judgment, the review panels), and asks what you want. Answer the questions. It writes `~/.cursor/rules/pstack-models.mdc`, a small rule every pstack skill reads.

The defaults are the same as the `large` budget. Code, prose, and judgment run on Opus 5.5 at `xhigh` reasoning, and the review panels and swarm workers run at `high`. No role runs above `xhigh`. Outside a review panel, Fable 5.2 runs only when you name it for a task or after two Opus 5.5 runs of the same brief fail their check. `medium` lowers each role one level and `small` lowers it two, and both spend fewer tokens.

You only override what you care about. A role with no line in the rule keeps the skill's default. To restore a default, delete that role's line. A rerun of `/setup-pstack` keeps any role whose model differs from the default. When a default changes, a rule written before the change still pins the old default until you run `/setup-pstack` again. That run replaces the old defaults it recognizes, tells you which, and keeps your other choices. To drop a pin it does not recognize, delete that role's line or the file first.

You might be wondering what happens if you use Auto. Set a role to `inherit-parent` or `auto` and pstack omits the subagent `model` field, so the subagent inherits your parent chat model. Both values mean the same thing, and neither is a model slug. For a panel role the value is a list, and one subagent runs per entry, so the list length sets the panel size. Setup also configures `swarm workers`, the default model for every `/swarm` worker unless a race names a model for each arm.

On OpenCode, `/setup-pstack` does not apply. It writes Cursor's `~/.cursor/rules/pstack-models.mdc` and reasons in Cursor slugs, so set `model: <provider>/<model>#<variant>` on `pslim-agent` yourself, or leave the line off so the subagent inherits the chat's model. Steps that need Cursor-only tooling, such as bugbot, automations, cloud agents, and `/loop`, have no OpenCode equivalent.

## Accept the verification offers, or don't

At the end of setup, `/setup-pstack` looks for a way to prove app behavior in your project, either a `verify-*` skill or an existing harness. If it finds neither, it offers once to generate one with [`/create-verification-skill`](../../skills/create-verification-skill/SKILL.md).

Say yes and it writes `.cursor/skills/verify-<app>/`, a project-local skill that teaches agents to drive your app the way a user does. It proves the skill works once before handing it over. Say no and setup moves on. You can run `/create-verification-skill` yourself any time. [Verify and ship](./06-verify-and-ship.md#create-a-project-verification-skill) covers it in depth.

If you're new to pstack, say yes. An agent that can check its own work keeps going until the check passes. An agent that can't hands every result back to you to check by hand. Of everything in this guide, the verification skill pays off the most.

Then setup offers once to generate a formal verification skill with [`/create-formal-verification-skill`](../../skills/create-formal-verification-skill/SKILL.md), unless the project already has one. Tests check the cases someone thought to write. A formal verification skill checks one key property of your code, such as "a job never runs twice at once", in every case within a stated scope. Say yes when your code has concurrency, retries, a state machine, or a parser, where a bug is costly and tests rarely reach the case that breaks. Otherwise say no, and run it later if that changes. [Verify and ship](./06-verify-and-ship.md#prove-a-property-with-a-formal-verification-skill) covers it.

After setup, start a new chat. The model rule applies to new sessions.

## Keep the cost in check

pstack spends extra tokens on subagents and review panels. That's the price of the rigor. To spend fewer:

- Rerun `/setup-pstack` and pick a smaller reasoning budget or cheaper models. A strong model in the main chat with cheaper, faster models in the code roles is a good split.
- Set a role to `auto` or `inherit-parent` so it runs on the chat's own model.
- Shorten a panel list. Each entry runs one subagent.
- Save `/pslim-mode` for work that needs rigor. A small, obvious edit doesn't.

## Run your first task

Pick something real but small, and describe it the way you'd describe it to a colleague:

```text
/pslim-mode add a --json flag to this command. text output stays byte-identical. verify both.
```

Watch the todo list. Its first items are the matched playbook's steps copied in, the Feature playbook for this prompt.

From here you can type normal follow-ups. To keep `/pslim-mode` on for the whole chat, pick it from the `/` menu with Option+Enter (Mac) or Alt+Enter (Windows) instead of Enter. That makes it a [Custom Mode](https://cursor.com/docs/skills#using-a-skill-as-a-custom-mode), which stays in context on every turn until you exit it. Custom Modes are available in the Agents Window and the CLI. Plain Enter attaches the skill to one message, and it fades as the chat moves on.

Next: [Route work through `/pslim-mode`](./02-pslim-mode.md).
