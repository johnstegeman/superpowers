---
name: jjify-maintenance
description: Use when syncing this jj fork with upstream obra/superpowers, rebasing the jjify patchset onto a new upstream revision, or reviewing upstream for new git references that need converting to jj equivalents.
---

# Maintaining the jjify Fork

This repo is a jj-adapted fork of [obra/superpowers](https://github.com/obra/superpowers).
The `jjify` bookmark marks the tip of a patch series that converts git references
to jj equivalents. The series lives on top of `main@origin`.

**Remotes:**
- `origin` → https://github.com/johnstegeman/superpowers (your fork, push target)
- `upstream` → https://github.com/obra/superpowers (canonical source, fetch only)
- `paulsmith` → https://github.com/paulsmith/superpowers (reference fork, kept for comparison)

## Sync Workflow

### 1. Fetch upstream

```bash
jj git fetch --remote upstream
```

### 2. Update main@origin to match upstream

```bash
jj bookmark set main -r main@upstream
jj git push -b main --remote origin
```

### 3. Rebase the jjify patches

Find the bottom patch and rebase the whole stack:

```bash
# Inspect the stack
jj log -r 'main@origin..jjify'

# Rebase from the bottom patch onto the new upstream
jj rebase -s rvknqlru -d main@origin
# (use the change ID of the bottom patch — currently rvknqlru)
```

Resolve any conflicts bottom-up (see Conflict Resolution Notes below).

### 4. Move the jjify bookmark to the new tip

```bash
jj bookmark set jjify -r <new-tip-change-id>
```

### 5. Push origin

```bash
jj git push -b main --remote origin
jj git push -b jjify --remote origin
```

## Parity Review

After rebasing, scan for new git references introduced by upstream that need conversion:

```bash
# Search for git commands in skill files
rg -i '\bgit (commit|add|push|pull|branch|worktree|stash|checkout|switch|merge|cherry-pick|rebase|reset|log|diff)\b' skills/
rg -i '\b(commit|committed) to git\b' skills/
rg -i '\bgit branches?\b' skills/
```

**Convert these patterns:**

| Pattern | Conversion |
|---------|-----------|
| `git commit`, `git add` | jj equivalents per jujutsu skill |
| `git branch` / "branches" | `jj bookmark` / "bookmarks" |
| `git worktree` / "worktrees" | `jj workspace` / "workspaces" |
| `git stash` | `jj new @-` pattern |
| `git log`, `git diff` | `jj log`, `jj diff` |
| `git checkout`, `git switch` | `jj edit` or `jj new` |
| `git merge` | `jj rebase` or `jj new A B` |
| `git cherry-pick` | `jj duplicate` |
| `git rebase` | `jj rebase` |
| `git reset` | `jj restore` or `jj abandon` |
| `git pull` | `jj git fetch --remote origin` |
| `git clone` | `jj git clone` |

**Ignore these (do not convert):**

| Pattern | Reason |
|---------|--------|
| `.gitignore` | jj uses gitignore natively |
| `jj git push`, `jj git fetch` | jj's git interop layer |
| GitHub / github.com references | The service, not the tool |
| `git rm --cached .jj-*` | Legitimate git index fixup (Step 7 of finishing-a-development-branch) |
| Red-flag lists warning against using git | Already correct |

## Conflict Resolution Notes

These decisions were made during the initial rebase onto v5.1.0 (2026-05-12).
Use them as guidance when the same files conflict again.

### Files deleted by upstream — honor the deletion

Upstream deleted these files before our initial rebase. Do not restore them:
- `.codex/INSTALL.md` — removed; codex docs reorganized elsewhere
- `docs/README.codex.md` — removed; same
- `tests/opencode/test-skills-core.sh` — removed; test infrastructure changed

### `.opencode/INSTALL.md` and `docs/README.opencode.md`

Upstream switched from a `git clone` manual install to an OpenCode plugin manifest
(`"plugin": ["superpowers@git+https://..."]`). Paul's patches applied git→jj changes
to the old manual install sections, which no longer exist in the current structure.

**Resolution:** Take upstream's structure. The plugin manifest install section does not
need jj conversion (it uses the OpenCode plugin manager, not git directly). Only the
`Updating` section is relevant — if it gains a `git pull`, convert it.

### `skills/finishing-a-development-branch/SKILL.md`

Paul's version simplified the skill structure. Upstream's v5.1.0 version is more elaborate
(adds environment detection, provenance-based workspace cleanup, harness-owned workspace
handling). The two versions are structurally incompatible.

**Resolution:** Take upstream's structure and apply jj conversions throughout:
- `git checkout` + `git pull` + `git merge` → `jj rebase -r <changes> -d trunk()`  + `jj bookmark set main`
- `git branch -d` → `jj bookmark delete` (or omit if not applicable)
- `git worktree` detection/removal → `jj workspace list` + `jj workspace forget` + `rm -rf`
- Add Paul's Step 7 (Post-Integration Cleanup) for `.jj-*` artifact cleanup
- Update Integration section: `using-git-worktrees` → `using-jj-workspaces`

### `skills/requesting-code-review/code-reviewer.md`

Paul's patch updated `git diff {BASE}..{HEAD}` → `jj diff --from {BASE} --to {HEAD}`.
Upstream substantially rewrote this template between Paul's base and v5.1.0.

**Resolution:** Take upstream's newer template; apply `git diff` → `jj diff` conversion.
Also rename section "Git Range to Review" → "Revision Range to Review".

### `lib/skills-core.js`

Paul's patch changed `checkForUpdates` from `git fetch && git status --porcelain` to
`jj git fetch --remote origin` + `jj log -r "main..main@origin"`.
Upstream added many new functions (`findSkillsInDir`, `resolveSkillPath`, etc.).

**Resolution:** Take upstream's full file; apply Paul's `checkForUpdates` conversion only.

### `plugin.json` fork metadata

Version scheme: `<upstream-version>jj` (e.g. `5.1.0jj`).
URLs: `https://github.com/johnstegeman/superpowers` (not paulsmith's).
Description prefix: `"jj-ified "`.
Keywords: add `"jj"` to the upstream list.

### `CLAUDE.md` and `AGENTS.md`

Paul replaced `CLAUDE.md` with `@AGENTS.md` and added `AGENTS.md` as a fork description.
Upstream keeps `CLAUDE.md` as contributor guidelines and `AGENTS.md` as a symlink → `CLAUDE.md`.

**Resolution:** Keep upstream's structure. `CLAUDE.md` = contributor guidelines.
`AGENTS.md` = symlink to `CLAUDE.md`. Do not replace with Paul's fork description.