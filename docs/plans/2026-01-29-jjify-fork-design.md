# Jjify Fork Design

Convert all git references in the superpowers plugin to jj (Jujutsu) equivalents,
maintained as a rebased patch series on top of upstream `obra/superpowers`.

## Fork Structure

- `main` bookmark tracks upstream's main (fetched from `obra/superpowers`)
- `jjify` bookmark marks the base point where patches begin
- Patches live as a linear commit series on top of `jjify`
- Each logical change is a separate commit for isolated rebase conflicts

### Upstream Sync Workflow

```bash
jj git fetch --remote upstream
jj rebase -b <first-patch> -d main
jj bookmark set jjify -r main
# Resolve conflicts, verify skills read correctly
```

## Patch Series

### Commit 1: Rename and rewrite `using-git-worktrees` → `using-jj-workspaces`

- Rename directory `skills/using-git-worktrees/` → `skills/using-jj-workspaces/`
- Rewrite `SKILL.md` content to teach jj workspaces instead of git worktrees
- Base content on Paul's existing `~/.claude/skills/using-jj-workspaces/SKILL.md`
- Update frontmatter name and description

### Commit 2: Convert `finishing-a-development-branch`

- `git merge` → `jj rebase`
- `git checkout` / `git pull` → `jj edit` / `jj git fetch`
- `git branch -d/-D` → `jj bookmark delete`
- `git worktree remove` → `jj workspace forget` + `rm -rf`
- `git worktree list` → `jj workspace list`
- `git merge-base` → jj revsets
- `git push -u origin` → `jj git push --remote origin -b <bookmark>`
- Rewrite the 4-option decision flow to use jj idioms throughout

### Commit 3: Convert `requesting-code-review`

- `git rev-parse HEAD~1` / `git rev-parse HEAD` → `jj log` with revsets
- `git log --oneline` → `jj log --limit N`
- Update SHA tracking to use jj change IDs

### Commit 4: Convert `writing-plans`

- `git add && git commit -m "..."` examples → `jj desc -m "..."` / `jj new -m "..."`
- Remove staging/index concepts from examples

### Commit 5: Update cross-references in other skills

Skills that reference `using-git-worktrees` by name:
- `brainstorming/SKILL.md`
- `subagent-driven-development/SKILL.md`
- `executing-plans/SKILL.md`
- `finishing-a-development-branch/SKILL.md`
- Any others found by grepping

Update all references to `using-jj-workspaces`.

### Commit 6: Convert `lib/skills-core.js`

- `checkForUpdates()`: replace `git fetch origin && git status --porcelain=v1 --branch`
  with `jj git fetch --remote origin` + revset-based divergence check
- Any other git commands in library code

### Commit 7: Tests, docs, README, install instructions

- Test scaffolding (`tests/subagent-driven-dev/*/scaffold.sh`): `git init` → `jj git init --colocate`
- `README.md`: installation instructions, git clone references
- `docs/README.opencode.md`, `docs/README.codex.md`: setup instructions
- `.opencode/INSTALL.md`, `.codex/INSTALL.md`
- `RELEASE-NOTES.md`: leave historical references as-is (they describe past events)

### Commit 8: Plugin metadata

- Update `plugin.json` repository URL to Paul's fork

## Post-Completion Cleanup

Remove redundant personal skills (these are outside the repo):
- `~/.claude/skills/using-jj-workspaces/` — fork now handles this
- `~/.claude/skills/using-git-worktrees/` — redirect shim no longer needed
- Keep `~/.claude/skills/jj-workflow/` — covers general jj usage beyond superpowers scope

## Conflict Risk Assessment

| Commit | Risk | Reason |
|--------|------|--------|
| 1 (worktrees → workspaces) | High | Directory rename + full rewrite |
| 2 (finishing-branch) | Medium | Substantial rewrite of active skill |
| 3 (requesting-review) | Low | Small, localized changes |
| 4 (writing-plans) | Low | Example code only |
| 5 (cross-references) | Low | String replacements |
| 6 (skills-core.js) | Low | Single function change |
| 7 (tests/docs) | Low-medium | Spread across many files |
| 8 (plugin metadata) | Trivial | One-line change |
