# AGENTS.md

Read this before changing anything. It explains what this repository is and what every upstream sync has to do.

## What this repo is

`pstack-slim` is a maintained fork of [`pstack`](https://github.com/cursor/plugins/tree/main/pstack), one plugin inside [`cursor/plugins`](https://github.com/cursor/plugins). The content was extracted with `git subtree split -P pstack`, so the history is pstack's own and none of the sibling plugins come along.

Three things are deliberately different from upstream:

1. **Names.** `poteto-mode`, `poteto-help`, and `poteto-agent` are `pslim-mode`, `pslim-help`, and `pslim-agent` here, so the fork's commands do not collide with upstream pstack. Do not restore the old names.
2. **Harness.** Upstream is written for Cursor. Here the skills target any coding agent that reads `SKILL.md` skills, with OpenCode as the primary target and Claude Code as a second adapter. Cursor-only tool names, paths, and features are moving out of the skills into `docs/harness/`; do not add new ones.
3. **Packaging.** The Cursor plugin manifest (`.cursor-plugin/`) is removed. Skills install standalone with `npx skills add shahrin014/pstack-slim -a <agent>`. `agents/pslim-agent.md` is installed separately, because the skills CLI ships skills only; that install also rewrites its Cursor `model:` and `is_background` fields.

The author's name, handle, and bio stay as written (`poteto`, `https://x.com/poteto`, "at Cursor"). This is a fork, not a rebrand.

## Branch and baseline

There is one branch: `main`. The fork, the adaptations, and the docs all land here, and it is what `git clone` and `npx skills add shahrin014/pstack-slim` resolve to.

The upstream baseline is a ref, not a branch. `refs/upstream/pstack` points at a commit whose tree is `cursor/plugins`'s `pstack/` directory at the last sync, laid out at this repo's root. It is the merge base for the next sync, and it cannot be checked out or committed to by accident.

Remotes: `origin` is `shahrin014/pstack-slim`, `upstream` is `cursor/plugins`.

## Syncing upstream

Run this from the repo root. It stages upstream's changes against the baseline in one merge, with no clone, no temp directory, and no history replay:

```bash
git fetch upstream main

if [ "$(git rev-parse refs/upstream/pstack^{tree})" != "$(git rev-parse upstream/main:pstack)" ]; then
  snap=$(git commit-tree "$(git rev-parse upstream/main:pstack)" -p refs/upstream/pstack \
    -m "chore: upstream pstack snapshot @ $(git rev-parse --short upstream/main)")
  git merge --no-ff --no-commit "$snap"
fi
```

If nothing is staged, upstream has not moved. Otherwise resolve and adapt (below), then finish the merge and move the baseline:

```bash
git commit
git update-ref refs/upstream/pstack "$snap"
```

Moving the baseline only after the adapted merge commits keeps the next sync's merge base correct. Renames are followed automatically, so an edit to a file this fork renamed lands in the `pslim-*` copy. Conflicts appear only where upstream and this fork changed the same lines: take upstream's intent into the `pslim-*` path, then re-apply the rules below.

`git subtree pull --prefix=pstack` does not apply, because the subtree sits at this repo's root. A plain `git merge upstream/main` would bring in every sibling plugin. `git merge -s subtree` cannot help either, because this history comes from a subtree split and shares no ancestors with upstream.

A fresh clone does not carry the baseline ref. Recover it from the last sync merge:

```bash
merge=$(git log --merges --grep='^chore: sync upstream pstack' -1 --format=%H)
git update-ref refs/upstream/pstack "$(git rev-parse "$merge^2^")"
```

Before any snapshot sync has happened, the baseline is the end of the imported upstream history, `3431628` in this repo.

## Adapting incoming changes (never skip)

Upstream changes arrive written for Cursor. Before committing a merge, walk this list over the diff:

1. **Tool names become operations.** No `Task`, `subagent_type`, `generalPurpose`, `readonly`, `run_in_background`, `environment: "cloud"`, `cloud_base_branch`, `file_attachments`, or `AskQuestion` in skill instructions, `agents/`, or `README.md` (TypeScript's `readonly` fields under `scripts/` are fine). Say spawn a subagent, read-only, in the background, isolated checkout, structured-choice question. Map each operation in `docs/harness/*.md`.
2. **Model slugs stay in `skills/setup-pstack/` and the pinned default in `agents/pslim-agent.md`.** Everywhere else uses role keys or model names (Opus 5.5, Fable 5.2, Gemini 4 Neon) and says "the harness's top reasoning level" instead of `xhigh`. Installers that reject Cursor slugs strip the agent's `model:` line.
3. **Paths get a harness-neutral home.** Three Cursor path families are still in the skills: the model rule (`~/.cursor/rules/pstack-models.mdc`), the skills roots (`.cursor/skills/`, `~/.cursor/skills/`, `plugins/cache/`), and transcripts (`~/.cursor/projects/<slug>/agent-transcripts/`). When one moves, update every skill that reads it; until then, add no more.
4. **Cursor-only features get a capability line.** Bugbot, Automations, Projects and cloud agents, Custom Modes, and `/loop` do not exist everywhere. Keep that behaviour behind the existing `Skipped in <harness>: <step>. <reason>.` convention.
5. **Preserve what is genuinely Cursor's.** Product names, `cursor.com` and `api2.cursor.sh` URLs, `.cursor/...` paths that name real Cursor files, `endCursor` identifiers in `scripts/watch-pr/`, and the author's bio. Never blind-replace `cursor`.
6. **Docs links in skills are absolute.** Skills install outside this repo, so repo-relative links (`../../docs/guide/...`) become `https://github.com/shahrin014/pstack-slim/blob/main/...`.
7. **Update `docs/harness/`.** Every harness-specific mapping lives there. When a change adds a tool-dependent step, add its row for each supported harness. Supported harnesses today: OpenCode (primary, `docs/harness/opencode.md`), Claude Code (`docs/harness/claude-code.md`), and Cursor as the upstream reference.

Commit the sync and the adaptation together on `main`, with the upstream commit named in the message:

```text
chore: sync upstream pstack (<upstream sha>)

<what changed> + how it was adapted
```

Commit on `main`.

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
