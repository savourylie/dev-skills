---
name: "cktk-upgrade"
description: "Use when the user explicitly asks to upgrade cktk to the latest version from GitHub. Prefer explicit invocation with $cktk-upgrade."
---

# Upgrade cktk

Pull the latest cktk skills from GitHub and update the local installation. Treat this as an explicit workflow skill. Do not trigger it just because a user mentioned upgrading in passing.

## Workflow

1. Detect the cktk install type by checking these locations in order:
   - `$HOME/.claude/plugins/marketplaces/cktk/.git` — if this directory exists, the install type is **plugin-marketplace** and the cktk root is `$HOME/.claude/plugins/marketplaces/cktk`.
   - The current git repo root (via `git rev-parse --show-toplevel`) with a remote URL containing `cktk` — if this matches, the install type is **git-clone** and the cktk root is the repo root.
   - If neither is found, report that no cktk git installation was found. Suggest installing via the Claude Code plugin marketplace or cloning from `https://github.com/savourylie/cktk`.

2. Record the current commit hash before upgrading:
   ```
   git -C <CKTK_DIR> rev-parse HEAD
   ```

3. Pull the latest version:
   ```
   cd <CKTK_DIR> && git fetch origin && git pull origin main
   ```
   If already up to date, say so and stop.

4. For **plugin-marketplace** installs only: sync the updated files to the plugin cache directory at `$HOME/.claude/plugins/cache/cktk/cktk/<version>/` (excluding `.git`).

5. Show the user what changed:
   - Previous commit vs new commit (short hashes)
   - `git log --oneline <OLD_HEAD>..HEAD`
   - For plugin-marketplace installs: note that a Claude Code restart may be needed.

## Safety Rules

- Do not modify any files outside the cktk installation directory.
- Do not force-push or reset. Use `git pull` only.
- If the pull fails due to local changes, report the error and stop instead of discarding changes.
