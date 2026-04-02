---
name: cktk-upgrade
description: "Upgrade cktk to the latest version. Triggers on: /cktk-upgrade, upgrade cktk, update cktk, get latest cktk"
user-invocable: true
allowed-tools: Bash(git:*), Bash(test:*), Bash(ls:*), Bash(cp:*), Bash(rm:*), Bash(mkdir:*), Bash(cat:*), Bash(rsync:*), Read
---

# Upgrade cktk

Pull the latest cktk skills from GitHub and update the local installation.

## Step 1: Detect install type

Run these checks to classify the installation:

1. Check if a plugin marketplace install exists:
   ```bash
   test -d "$HOME/.claude/plugins/marketplaces/cktk/.git" && echo "yes" || echo "no"
   ```

2. If `yes`, get the plugin marketplace path and cache path:
   ```bash
   cd "$HOME/.claude/plugins/marketplaces/cktk" && pwd
   ```
   ```bash
   ls -d "$HOME/.claude/plugins/cache/cktk/cktk/"* 2>/dev/null | head -1 || echo "N/A"
   ```
   Set **INSTALL_TYPE** to `plugin-marketplace` and **CKTK_DIR** to the marketplace path.

3. Else, check if the current working directory is a cktk git clone:
   ```bash
   git -C "$(git rev-parse --show-toplevel 2>/dev/null)" remote get-url origin 2>/dev/null | grep -q 'cktk' && echo "yes" || echo "no"
   ```
   If `yes`, get the repo root:
   ```bash
   git rev-parse --show-toplevel
   ```
   Set **INSTALL_TYPE** to `git-clone` and **CKTK_DIR** to the repo root.

4. If neither is found, report that no cktk git installation was found and stop. Suggest the user install via `/plugin marketplace add savourylie/cktk` then `/plugin install cktk@savourylie`, or clone from `https://github.com/savourylie/cktk`.

## Step 2: Record current state

Run `git -C <CKTK_DIR> rev-parse HEAD` and save the output as **OLD_HEAD**.

## Step 3: Pull latest

Run:

```bash
cd <CKTK_DIR> && git fetch origin && git pull origin main
```

If the pull reports "Already up to date", tell the user cktk is already on the latest version and stop.

### Sync plugin cache (plugin-marketplace only)

If **INSTALL_TYPE** is `plugin-marketplace` and the plugin cache path detected in Step 1 is not `N/A`, sync the updated files into the cache:

```bash
rsync -a --delete --exclude '.git' <CKTK_DIR>/ <CACHE_PATH>/
```

## Step 4: Report

Run `git -C <CKTK_DIR> rev-parse --short HEAD` to get the new short hash and `git -C <CKTK_DIR> log --oneline <OLD_HEAD>..HEAD` to list changes.

Report to the user:
- Previous version: `<OLD_HEAD short>`
- Updated to: `<NEW_HEAD short>`
- Changes pulled (the log output)
- If **INSTALL_TYPE** is `plugin-marketplace`: note that a Claude Code restart may be needed for the updated skills to take effect.

Do not send any other text or messages besides the tool calls and the final report.
