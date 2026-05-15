---
name: "merge-worktree"
description: "Use when the user explicitly asks to merge one or more ticket worktrees back into their base branch and clean them up (the cleanup half of $create-worktree). Pass `no-cleanup` to keep the local branch around after merging (worktree directory is still removed). Prefer explicit invocation with $merge-worktree."
---

# Merge Worktrees Back Into Base

Close the loop on `$create-worktree`: merge each ticket's branch into its base, then remove the worktree directory and delete the local branch. Detect already-merged branches (typical after a GitHub PR merge) and skip straight to cleanup so we don't create a redundant merge commit. Treat this as an explicit workflow skill because it mutates the working tree, deletes branches, and creates merge commits.

Parse any text that follows the skill invocation as arguments. Accept ticket identifiers like `TICKET-002`, `002`, `#002`, or `2`. Accept one optional non-ticket, non-flag token as the base branch. Accept the literal flag `no-cleanup` to preserve the local branch after merging. Default base is `main`.

## Phase 1: Parse Arguments

1. Split the argument string on whitespace.
2. Classify each token:
   - **Ticket number** if it matches `^\d+$`, `^#\d+$`, or `^TICKET-\d+$` (case-insensitive). Normalize to a 3-digit zero-padded number.
   - **`no-cleanup` flag** if it matches `^no-cleanup$` (case-insensitive). When set, the local branch is preserved after merging (the worktree directory is still removed). Duplicates are ignored. Does not consume the base-branch slot.
   - **Base branch** otherwise. At most one such token; if two or more non-ticket, non-flag tokens, report the conflict and stop.
3. Require at least one ticket number; if none, ask the user and stop.
4. Default base is `main` if no base-branch token was given.

Examples:
- `$merge-worktree 7` → tickets `[007]`, base `main`, no-cleanup `false`
- `$merge-worktree 7 8 9` → tickets `[007, 008, 009]`, base `main`, no-cleanup `false`
- `$merge-worktree 7 dev` → tickets `[007]`, base `dev`, no-cleanup `false`
- `$merge-worktree 7 no-cleanup` → tickets `[007]`, base `main`, no-cleanup `true`
- `$merge-worktree 7 dev no-cleanup` → tickets `[007]`, base `dev`, no-cleanup `true`

## Phase 2: Resolve Repo Root and Tickets

1. Confirm we're in a git repo and resolve the **main** repo root so the skill works the same from any worktree:
   ```
   MAIN_ROOT=$(dirname "$(git rev-parse --path-format=absolute --git-common-dir)")
   ```
2. For each ticket number `NNN`, glob `$MAIN_ROOT/docs/tickets/NNN-*.md`.
   - No match → report and stop the whole batch (typo in one number should not result in partial cleanup).
   - Multiple matches → stop and report.
3. Slug = the part of the filename between the `NNN-` prefix and the `.md` suffix. Filename slug is authoritative.
4. Per ticket: worktree path `$MAIN_ROOT/.worktrees/NNN-<slug>`, branch `ticket-NNN-<slug>`.

## Phase 3: Pre-flight Checks

Run once before touching any ticket. Any failure stops the whole batch.

1. **CWD must not be inside a target worktree.** Compare `git rev-parse --show-toplevel` against each target worktree path. If it matches, refuse and tell the user to `cd "$MAIN_ROOT"` — removing the cwd would orphan their shell.
2. **Main checkout must be clean.** `git -C "$MAIN_ROOT" status --porcelain` must be empty. The merge lands on the main checkout's HEAD, so a dirty main checkout puts the user's work at risk.
3. **Fetch the base** with `git -C "$MAIN_ROOT" fetch origin <base>`. If `origin` is missing or the fetch fails, fall back to local `<base>` and warn that the result may be stale.
4. **Resolve base ref**: prefer `origin/<base>`; fall back to local `<base>`. If neither exists, report and stop.
5. **Switch and fast-forward.** Switch the main checkout to local `<base>` (creating it from `origin/<base>` if needed), then `git merge --ff-only origin/<base>` to bring it up to date. If the FF fails (local base diverged), abort with a clear error — that needs human judgment.

## Phase 4: Per-Ticket Merge and Cleanup

Process tickets in order. One ticket's failure must not abort the others.

For each ticket:

1. **Already cleaned up?** If neither the worktree path nor the branch exists, record as no-op and continue.
2. **Worktree clean?** `git -C <worktree_path> status --porcelain` must be empty. Otherwise skip with a clear error and continue — never silently lose uncommitted work.
3. **Already merged?** `git -C "$MAIN_ROOT" merge-base --is-ancestor <branch> <base-ref>`. Exit 0 → skip merge, go to step 5.
4. **Merge.** `git -C "$MAIN_ROOT" merge --no-ff -m "Merge <branch> into <base>" <branch>`. `--no-ff` keeps each ticket revertable as a unit.
   - On conflict: `git merge --abort`, record the conflicted files, skip cleanup for this ticket, continue.
   - On other failure: record the error, skip cleanup, continue.
5. **Cleanup.**
   - `git -C "$MAIN_ROOT" worktree remove <worktree_path>` (run `git worktree prune` and retry once if it complains the dir is missing on disk).
   - If `no-cleanup` was passed, stop here and record `branch retained (no-cleanup)`. Otherwise `git -C "$MAIN_ROOT" branch -D <branch>` — force-delete is safe and intentional because step 3's `--is-ancestor` check or step 4's successful merge has already established the branch's content is in the base. This avoids the common GitHub squash-/rebase-merge failure where `git branch -d` would refuse.

## Phase 5: Report

Print a per-ticket summary grouped by outcome (merged-and-cleaned, already-merged-and-cleaned, branch-retained-by-flag, skipped, conflict, error), and the final HEAD of the main checkout. The `branch retained (no-cleanup)` label is informational — the user asked for it explicitly via the flag. Do not push to origin, do not delete remote branches, do not commit anything beyond the merge commits themselves.

## Safety Rules

- Stop the whole batch up front if any ticket file is missing or any pre-flight check fails. Partial cleanup hides typos and leaves an inconsistent state.
- `git branch -D` is the intended branch-delete in Phase 4 step 5. It is safe there because step 3 (`--is-ancestor`) or step 4 (`merge --no-ff`) has already established the branch's content is in the base. Do not use `-D` anywhere else in the flow.
- Never run `git worktree remove --force` as a fallback for a normal failure — it can lose uncommitted work without warning. Use `git worktree prune` + retry instead.
- Never push, never delete remote branches.
- Never modify the worktree's contents — only inspect status, then merge into base from the main checkout.
