---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for integration, PR, or cleanup
---

# Finishing a Development Branch

## Overview

Guide completion of development work by presenting clear options and handling chosen workflow.

**Core principle:** Verify tests → Detect environment → Present options → Execute choice → Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / uv run pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with integration/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Detect Environment

**Determine workspace state before presenting options:**

```bash
# List all jj workspaces and their paths
jj workspace list --no-pager
```

This determines which menu to show and how cleanup works:

| State | Menu | Cleanup |
|-------|------|---------|
| Default workspace (repo root) | Standard 4 options | No workspace to clean up |
| Named workspace under `.workspaces/` | Standard 4 options | Provenance-based (see Step 6) |
| Named workspace (harness-owned) | Reduced 3 options | No cleanup (externally managed) |

A workspace is "harness-owned" if it was not created by the `using-jj-workspaces` skill (i.e., not under `.workspaces/`).

### Step 3: Determine Base Revision

```bash
# Check where your work diverged from trunk
jj log

# Identify the base - typically trunk() or the 'main' bookmark
jj log -r "trunk()"
jj log -r "bookmarks(exact:main)"
```

Or ask: "This work is based on main — is that correct?"

### Step 4: Present Options

**Default workspace or provenance-owned workspace — present exactly these 4 options:**

```
Implementation complete. What would you like to do?

1. Integrate changes into main locally
2. Push and create a Pull Request
3. Keep the changes as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Harness-owned workspace — present exactly these 3 options:**

```
Implementation complete. You're in an externally managed workspace.

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**Don't add explanation** - keep options concise.

### Step 5: Execute Choice

#### Option 1: Integrate Locally

```bash
# Rebase your changes onto trunk if needed
jj rebase -r <your-changes> -d 'trunk()'

# Move the main bookmark forward
jj bookmark set main -r <your-final-change>
# Or if the user has the 'tug' alias:
jj tug

# Verify tests on integrated result
<test command>

# Verify
jj log -r 'trunk()..@'

# Start fresh
jj new
```

Then: Cleanup workspace (Step 6), then Post-Integration Cleanup (Step 7)

#### Option 2: Push and Create PR

```bash
# Push bookmark to remote
jj bookmark set <bookmark-name> -r @
jj git push -b <bookmark-name>
```

Then: Create PR. Do NOT clean up workspace (user needs it for PR iteration).

#### Option 3: Keep As-Is

No action needed. Inform user where to find the work:

```
Changes preserved. Workspace/changes remain in current state.
```

#### Option 4: Discard

**Require typed confirmation:**

```
This will permanently discard all work in this change. Type "discard" to confirm:
```

Wait for user to type "discard".

```bash
# Abandon the change(s)
jj abandon <change-id>
```

Then: Cleanup workspace (Step 6).

### Step 6: Cleanup Workspace

**Only runs for Options 1 and 4.** Options 2 and 3 always preserve the workspace.

```bash
# Record workspace path before changing directories
WORKSPACE_PATH=$(pwd)
WORKSPACE_NAME=$(basename "$WORKSPACE_PATH")
```

**If in default workspace:** No workspace to clean up. Done.

**If workspace path is under `.workspaces/`:** Superpowers created this workspace — we own cleanup.

```bash
# Navigate to the repo root (outside the workspace)
cd "$(jj root --no-pager)"

# Forget the workspace record first
jj workspace forget "$WORKSPACE_NAME"

# Remove the directory (verify path first!)
rm -rf "$WORKSPACE_PATH"
```

**Otherwise:** The host environment (harness) owns this workspace. Do NOT remove it. If your platform provides a workspace-exit tool, use it. Otherwise, leave the workspace in place.

### Step 7: Post-Integration Cleanup

After integration, especially if there were conflicts:

```bash
# Clean any stale jj artifacts from git's index
# (jj uses git for storage, conflict markers can get stuck)
git status --porcelain | grep -E "\.jj-" && git rm --cached .jj-* 2>/dev/null

# Verify environment still works
direnv allow 2>/dev/null

# Run the actual app, not just tests
# (tests use temp dirs, won't catch missing production paths)
make serve  # or equivalent
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Tests fail | Fix before proceeding |
| All good, want to land now | Option 1 (Integrate) |
| Need PR review | Option 2 (Push) |
| Not ready to land | Option 3 (Keep) |
| Throw it away | Option 4 (Discard) |

## Common Mistakes

**Skipping test verification**
- **Problem:** Integrate broken code, create failing PR
- **Fix:** Always verify tests before offering options

**Open-ended questions**
- **Problem:** "What should I do next?" is ambiguous
- **Fix:** Present exactly 4 structured options (or 3 for harness-owned workspaces)

**Cleaning up workspace for Option 2**
- **Problem:** Remove workspace user needs for PR iteration
- **Fix:** Only cleanup for Options 1 and 4

**Forgetting workspace before directory removal**
- **Problem:** jj workspace record persists after directory removed
- **Fix:** Run `jj workspace forget` before `rm -rf`

**Cleaning up harness-owned workspaces**
- **Problem:** Removing a workspace the harness created causes phantom state
- **Fix:** Only clean up workspaces under `.workspaces/`

**No confirmation for discard**
- **Problem:** Accidentally delete work
- **Fix:** Require typed "discard" confirmation

**Using `git merge`, `git checkout`, `git branch`**
- **Problem:** These corrupt jj state in a jj repo
- **Fix:** Use jj equivalents: `jj rebase`, `jj edit`, `jj bookmark`

## Red Flags

**Never:**
- Proceed with failing tests
- Integrate without verifying tests on result
- Delete work without confirmation
- Use `git merge`, `git checkout`, `git branch` — use jj equivalents
- Delete workspace directory before `jj workspace forget`
- Cleanup workspace when PR is pending review (Option 2)
- Cleanup harness-owned workspaces

**Always:**
- Verify tests before offering options
- Detect environment before presenting menu
- Present exactly 4 options (or 3 for harness-owned workspaces)
- Get typed confirmation for Option 4
- Clean up workspace for Options 1 & 4 only
- Use meaningful descriptions on changes
- Confirm workspace path before `rm -rf`
- Run actual application after integration, not just tests

## Integration

**Called by:**
- **subagent-driven-development** (Step 7) - After all tasks complete
- **executing-plans** (Step 5) - After all batches complete

**Pairs with:**
- **using-jj-workspaces** - Cleans up workspace created by that skill
