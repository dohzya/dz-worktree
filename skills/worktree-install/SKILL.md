---
name: worktree-install
description: >
  Install dz-worktree for the current user using the bundled interactive
  scripts/install helper, then initialize project Git hooks with
  git-worktree-init. Use when the user asks to install or set up dz-worktree,
  Git worktree hooks, shell functions, or the worktree skills for Claude and/or
  Codex.
---

# Worktree Install

Use the bundled installer instead of recreating installation steps manually.

## Procedure

1. Locate the dz-worktree root. From this skill, it is two directories up and
   must contain `scripts/install`, `git-hooks/`, `shell/`, and `skills/`.
2. For user-level setup, run the interactive installer. It detects installed
   shells, Claude/Codex homes, and `init.templateDir`, then asks before writing
   shell files, skills, or template hooks.

```bash
<dz-worktree>/scripts/install
```

3. For project hook setup after shell installation, use:

```bash
git-worktree-init --dry-run
git-worktree-init
```

`git-worktree-init` uses Git's effective hook path for the current worktree, so
it handles normal common-dir hooks, `core.hooksPath`, and worktree-specific hook
config.

4. For non-interactive preview or CI-style usage, use:

```bash
<dz-worktree>/scripts/install --yes --dry-run
```

5. Do not pass `--force` unless the user explicitly approves overwriting
   existing generated files, hooks, or skills.

After installing Codex skills, tell the user to restart Codex.
