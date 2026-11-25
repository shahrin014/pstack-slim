# AGENTS.md

Read this before changing anything. It explains what this repository is and what every upstream sync has to do.

## What this repo is

`pstack-slim` is a maintained fork of [`pstack`](https://github.com/cursor/plugins/tree/main/pstack), one plugin inside [`cursor/plugins`](https://github.com/cursor/plugins). The content was extracted with `git subtree split -P pstack`, so the history is pstack's own and none of the sibling plugins come along.

Three things are deliberately different from upstream:

1. **Names.** `poteto-mode`, `poteto-help`, and `poteto-agent` are `pslim-mode`, `pslim-help`, and `pslim-agent` here, so the fork's commands do not collide with upstream pstack. Do not restore the old names.
2. **Harness.** Upstream is written for Cursor. Here the skills target any coding agent that reads `SKILL.md` skills, with OpenCode as the primary target and Claude Code as a second adapter. Cursor-only tool names, paths, and features belong in `docs/harness/`, never in a skill.
3. **Packaging.** The Cursor plugin manifest (`.cursor-plugin/`) is removed. Skills install standalone with `npx skills add shahrin014/pstack-slim -a <agent>`. `agents/pslim-agent.md` is installed separately, because the skills CLI ships skills only; that install also rewrites its Cursor `model:` and `is_background` fields.

The author's name, handle, and bio stay as written (`poteto`, `https://x.com/poteto`, "at Cursor"). This is a fork, not a rebrand.

## Branches

| Branch     | Holds                                                             | Rule                                                            |
| ---------- | ----------------------------------------------------------------- | --------------------------------------------------------------- |
| `main`     | The fork. All work, adaptation, and docs land here.               | Commit here. This is the branch formerly called `pslim`.        |
| `pstack`   | The upstream split at the last sync point, matching `cursor/plugins/pstack`. | Never commit. It is the merge base for the next sync. |

Remotes: `origin` is `shahrin014/pstack-slim`, `upstream` is `cursor/plugins`. `refs/upstream/pstack` caches the latest split tip.

## Syncing upstream

Run this from the repo root. A split needs the `pstack/` prefix in a worktree, so it is computed in a scratch clone:

```bash
tmp=$(mktemp -d)
git clone -q https://github.com/cursor/plugins.git "$tmp/plugins"
git -C "$tmp/plugins" subtree split -P pstack -b pstack-only
git fetch "$tmp/plugins" pstack-only:refs/upstream/pstack
rm -rf "$tmp"

git merge refs/upstream/pstack
```

The merge brings in upstream pstack. It conflicts wherever upstream edited a path this fork renamed (`poteto-*` upstream, `pslim-*` here). Resolve by taking upstream's content at the `pslim-*` path, then re-applying the rules below.

Only move the baseline when you want a new one:

```bash
git branch -f pstack refs/upstream/pstack
```

## Adapting incoming changes (never skip)

Upstream changes arrive written for Cursor. Before committing a merge, walk this list over the diff:

1. **Tool names become operations.** No `Task`, `subagent_type`, `generalPurpose`, `readonly`, `run_in_background`, `environment: "cloud"`, `cloud_base_branch`, `file_attachments`, or `AskQuestion` in `skills/`, `agents/`, or `README.md`. Say spawn a subagent, read-only, in the background, isolated checkout, structured-choice question. Map each operation in `docs/harness/*.md`.
2. **Model slugs stay in `skills/setup-pstack/`.** Everywhere else uses role keys or model names (Opus 5.5, Fable 5.2, Gemini 4 Neon) and says "the harness's top reasoning level" instead of `xhigh`.
3. **Paths get a harness-neutral home.** `~/.cursor/rules/pstack-models.mdc` is the one Cursor path still in the skills; when it moves, update every skill that reads it. `.cursor/skills/` becomes the harness's skills root. Transcript locations come from the harness adapter.
4. **Cursor-only features get a capability line.** Bugbot, Automations, Projects and cloud agents, Custom Modes, and `/loop` do not exist everywhere. Keep that behaviour behind the existing `Skipped in <harness>: <step>. <reason>.` convention.
5. **Preserve what is genuinely Cursor's.** Product names, `cursor.com` and `api2.cursor.sh` URLs, `.cursor/...` paths that name real Cursor files, `endCursor` identifiers in `scripts/watch-pr/`, and the author's bio. Never blind-replace `cursor`.
6. **Docs links in skills are absolute.** Skills install outside this repo, so repo-relative links (`../../docs/guide/...`) become `https://github.com/shahrin014/pstack-slim/blob/main/...`.
7. **Update `docs/harness/`.** Every harness-specific mapping lives there. When a change adds a tool-dependent step, add its row for each supported harness.

Commit the sync and the adaptation together on `main`, with the upstream commit named in the message:

```text
chore: sync upstream pstack (<upstream sha>)

<what changed> + how it was adapted
```

Never commit adaptation work to `pstack`.

## Verify before commit

```bash
# no upstream names, no Cursor protocol leaked into skills
git grep -nE 'poteto-(mode|agent|help)|subagent_type|generalPurpose|run_in_background|environment: "cloud"|AskQuestion|\bTask\b' -- skills agents README.md docs/guide

# model slugs outside the model-config skill
git grep -nE 'claude-[a-z0-9]+-[0-9]|gemini-[0-9]|grok-[0-9]' -- skills ':!skills/setup-pstack'
```

Both should print nothing. Then check that relative links still resolve:

```bash
python3 - <<'PY'
import re, os, glob
bad = []
for f in glob.glob('**/*.md', recursive=True):
    for m in re.finditer(r'\]\(([^)\s]+)\)', open(f, encoding='utf-8').read()):
        p = m.group(1).split('#')[0]
        if p and '://' not in p and not p.startswith('mailto:'):
            if not os.path.exists(os.path.normpath(os.path.join(os.path.dirname(f), p))):
                bad.append((f, p))
print(len(bad), 'broken relative links', bad[:10])
PY
```

## Conventions

- Prose is lowercase and direct, matching upstream.
- A skill is self-contained: everything it needs sits in its own folder, and paths inside it are relative to `SKILL.md`.
- Keep diffs minimal. Do not reformat lines you are not changing; that makes the next upstream merge harder.
- Bump `version` in `plugin.json` when behaviour changes.
- `LICENSE` stays MIT, and `README.md` keeps upstream's credit lines.
