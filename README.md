# dz-worktree

Git worktree toolkit for bare-repo setups. Replaces worktrunk with native git
hooks and fish shell functions.

## Prerequisites

- Git 2.36+ (for `git hook run`)
- bash, zsh, or fish
- fzf
- Optional: mise, task (Taskfile)

## Layout

```
dz-worktree/
├── git-hooks/
│   ├── post-checkout          # Bootstrap new worktrees from primary
│   └── pre-remove-worktree    # Cleanup before worktree removal
├── scripts/
│   └── install                # Interactive user installer
├── shell/
│   ├── fish/git-worktree.fish # Shell functions (fish)
│   ├── bash/git-worktree.bash # Shell functions (bash)
│   └── zsh/git-worktree.zsh   # Shell functions (zsh)
├── skills/
│   ├── worktree-install/
│   │   └── SKILL.md           # Skill: install dz-worktree into a repo
│   └── worktree-sync/
│       └── SKILL.md           # Skill: diff & sync bootstrapped worktree files
└── README.md
```

## Setup

### 1. User install

Install shell functions and user-level agent skills:

```bash
~/bin/share/dz-worktree/scripts/install
```

The installer detects installed shells, Claude/Codex homes, and
`init.templateDir`. It asks before installing:

- shell files for detected shells
- Claude user skills into `${CLAUDE_HOME:-~/.claude}/skills/`
- Codex user skills into `${CODEX_HOME:-~/.codex}/skills/`
- hooks into the configured Git template directory, when `init.templateDir` is set

Shell files are generated as standalone copies under your shell config. They do
not depend on this `dz-worktree` checkout. Their `git-worktree-init` function
embeds the hook contents directly.

Git hooks are not installed globally by default. Hooks are project-specific, and
a global `post-checkout` hook would run for every repository on the machine.
After user install, run `git-worktree-init` inside any project that should use
the hooks.

### 2. Project install

From the target project:

```bash
git-worktree-init
```

This installs Git hooks into the repo's effective hooks directory as resolved by
Git for that worktree. This covers normal common-dir hooks, `core.hooksPath`,
and worktree-specific hook config.

Useful options:

```bash
~/bin/share/dz-worktree/scripts/install --dry-run
~/bin/share/dz-worktree/scripts/install --yes
git-worktree-init --dry-run
git-worktree-init /path/to/project
```

`--dry-run` previews all detected install actions without prompting or writing.

Restart Codex after installing Codex skills.

### 3. Git hooks

Copy hooks into your bare repo:

```bash
cp git-hooks/post-checkout  <bare-repo>/hooks/post-checkout
cp git-hooks/pre-remove-worktree <bare-repo>/hooks/pre-remove-worktree
chmod +x <bare-repo>/hooks/post-checkout <bare-repo>/hooks/pre-remove-worktree
```

Then edit `post-checkout` to configure:

- **Files to copy** — list of untracked files to bootstrap into new worktrees
- **Directories to copy** — `.claude/`, `docs/`, or whatever your project needs
- **Setup commands** — post-bootstrap commands (`mise trust`, `task setup`, etc.)
- **.env sed pattern** — adjust if your `.env` first line uses a different format

And edit `pre-remove-worktree` to configure:

- **Cleanup commands** — uncomment/add project-specific cleanup (worklog, etc.)

The primary worktree is detected automatically in both hooks (same logic as the
shell functions).

### 4. Shell functions

Source or symlink the file matching your shell:

```bash
# fish
ln -s ~/bin/share/dz-worktree/shell/fish/git-worktree.fish ~/.config/fish/conf.d/git-worktree.fish

# bash — add to ~/.bashrc
source ~/bin/share/dz-worktree/shell/bash/git-worktree.bash

# zsh — add to ~/.zshrc
source ~/bin/share/dz-worktree/shell/zsh/git-worktree.zsh
```

The primary worktree is detected automatically (no configuration needed):
non-bare repos use `git rev-parse --show-toplevel`, bare repos find the
worktree on the default branch (`origin/HEAD` → `init.defaultBranch` → `main`).

### 5. Agent skills (optional)

The installer can install skills automatically. Manual installation is also
possible:

```bash
cp -R skills/worktree-sync <project>/.claude/skills/worktree-sync
cp -R skills/worktree-install <project>/.claude/skills/worktree-install
cp -R skills/worktree-sync ${CLAUDE_HOME:-~/.claude}/skills/worktree-sync
cp -R skills/worktree-install ${CLAUDE_HOME:-~/.claude}/skills/worktree-install
cp -R skills/worktree-sync ${CODEX_HOME:-~/.codex}/skills/worktree-sync
cp -R skills/worktree-install ${CODEX_HOME:-~/.codex}/skills/worktree-install
```

`worktree-sync` derives syncable targets from the installed `post-checkout`
hook, so update the hook when project bootstrap rules change.

## Usage

```bash
# Create a worktree (tracks remote branch if available)
git-worktree-create feat/my-feature
git-worktree-create feat/my-feature ../my-feature
git-worktree-create -b chore/fix-truc ../fix-truc
git-worktree-create --help

# Initialize current repo with dz-worktree hooks
git-worktree-init
git-worktree-init --dry-run

# Switch between worktrees
git-worktree-switch                      # fzf picker
git-worktree-switch main                 # by branch name
git-worktree-switch feat-my-feature      # by directory name
git-worktree-switch /absolute/path       # by full path

# Remove a worktree (runs pre-remove hook, cd's out if needed)
git-worktree-remove                      # fzf picker
git-worktree-remove feat/my-feature      # by branch name
git-worktree-remove feat-my-feature      # by directory name
git-worktree-remove -f feat-my-feature   # force (untracked files)

# Sync check (agent skill)
# /worktree-sync              — from a worktree, compare with primary
# /worktree-sync my-feature   — from primary, compare with a worktree
```

The argument to `switch` and `remove` can be a branch name or a worktree
directory name. In the rare case where the same input matches both and they
point to different worktrees, the command asks which one you mean.

## How it works

### post-checkout hook

Git fires `post-checkout` on every `git worktree add`. The hook detects
worktree creation by checking if `$1` (previous HEAD) is the null SHA
(`000...0`) and `$3` is `1` (branch checkout, not file checkout). It then
copies untracked project files from the primary worktree and runs setup.

On any branch checkout (not just creation), it refreshes the terminal tab title.

### pre-remove-worktree hook

Not a native git hook — git has no automatic hook for worktree removal.
Called manually via `git hook run --allow-unknown-hook-name` or automatically
by `git-worktree-remove`. Runs cleanup before the worktree directory is deleted.

### Shell functions

Must be shell functions (not scripts) because `cd` in a subprocess doesn't
affect the parent shell. `git-worktree-create` reproduces the `--guess`
behavior of `git switch`: if a branch exists on exactly one remote but not
locally, it creates a tracking branch.

`__git_wt_resolve` resolves an argument to a worktree path by matching against
both directory basenames and checked-out branch names from `git worktree list`.
Used by `git-worktree-switch` and `git-worktree-remove`.

`__git_wt_primary` finds the primary worktree without configuration: in
non-bare repos it's the toplevel, in bare repos it's whichever worktree has the
default branch checked out.
