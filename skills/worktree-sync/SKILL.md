---
name: worktree-sync
description: >
  Check for un-synced changes to files bootstrapped into Git worktrees by the
  project's post-checkout hook. Reports diffs and offers to copy changes in
  either direction. Works from a worktree or from the primary worktree when a
  worktree name/path is provided. Use when: (1) before removing a worktree,
  (2) user mentions "sync", "worktree-sync", "worktree sync", (3) user wants
  to check whether bootstrapped worktree files diverged from the primary
  worktree. Argument: optional worktree name or path.
---

# Worktree Sync

Compare files that the project bootstraps from the primary worktree into new
worktrees. The source of truth is the installed `post-checkout` hook, not this
skill.

## Procedure

### 1. Resolve paths

Derive the repo root, installed hook path, and primary worktree path. Use the
same primary detection logic as the hook:

```bash
common_dir="$(git rev-parse --git-common-dir)"
repo_root="$(cd "$common_dir/.." && pwd)"
hook_path="$common_dir/hooks/post-checkout"

default_branch="$(git for-each-ref --format='%(symref:short)' 'refs/remotes/*/HEAD' | awk 'NF { print; exit }')"
default_branch="${default_branch#*/}"
[[ -z "$default_branch" ]] && default_branch="$(git --git-dir="$common_dir" symbolic-ref --short HEAD 2>/dev/null)"
[[ -z "$default_branch" ]] && default_branch="$(git config init.defaultBranch 2>/dev/null)"
[[ -z "$default_branch" ]] && default_branch=main
primary="$(
  git worktree list --porcelain |
    awk -v b="refs/heads/$default_branch" '
      $1 == "worktree" { path = substr($0, 10) }
      $1 == "branch" && $2 == b { print path; exit }
    '
)"
if [[ -z "$primary" && "$(git rev-parse --is-bare-repository 2>/dev/null)" == false ]]; then
  primary="$(git rev-parse --show-toplevel)"
fi
```

Determine worktree path based on invocation context:

- **Argument provided** (e.g. `/worktree-sync my-feature`): resolve `wt_path`
  from the argument — treat it as a name under `$repo_root/` or as an absolute
  path if it starts with `/`.
- **No argument, cwd is not primary**: use `pwd` as the worktree.
- **No argument, cwd is primary**: list worktrees with `git worktree list`,
  ask the user to pick one.

Abort if the resolved worktree IS the primary (nothing to compare).

### 2. Derive sync targets

Read `hook_path` and identify every path that the hook bootstraps from the
primary worktree into the new worktree:

- copied single files
- copied directories
- generated symlinks whose source is bootstrapped by the hook
- copied files with deterministic post-copy transformations

Ignore setup commands and cosmetic commands. Do not invent target paths that are
not represented in the hook. If the hook is missing or ambiguous, say what is
unclear and ask the user before syncing.

### 3. Diff

For each derived target, compare the worktree path with the primary path.
Use recursive names-only diff for directories and quiet diff for files:

```bash
diff -rq "$wt_path/$target" "$primary/$target" 2>/dev/null
diff -q "$wt_path/$target" "$primary/$target" 2>/dev/null
```

Respect exclusions or transformations encoded in the hook. For transformed
files, compare the meaningful content and explain the expected transformation
instead of treating it as an unexplained drift.

### 4. Report

If no differences: say "In sync" and stop.

Otherwise, for each differing file show a short `diff -u` excerpt and classify:

- **worktree → primary**: worktree has changes worth syncing back
- **primary → worktree**: primary was updated since bootstrap (worktree is stale)
- **only in worktree**: new file created in worktree
- **only in primary**: new file created in primary since bootstrap

### 5. Sync

Ask the user which direction to sync (per file or batch):

- **worktree → primary**: `cp` from worktree to primary (needs explicit approval)
- **primary → worktree**: `cp` from primary to worktree
- **skip**: leave as-is

Never overwrite without user confirmation.
