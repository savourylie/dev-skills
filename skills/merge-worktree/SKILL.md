---
name: merge-worktree
description: "Merge one or more ticket worktrees back into their base branch, then remove the worktree directory and delete the local branch. The cleanup half of /create-worktree. Detects already-merged branches (e.g., merged via GitHub PR) and just cleans up in that case. Triggers on: /merge-worktree, merge worktree, finish ticket worktree, land worktree, clean up worktree after ticket, remove worktree after merging, done with ticket worktree, ticket worktree finished"
user-invocable: true
---

**Argument:** `$ARGUMENTS`

Close the loop on `/create-worktree`. Merge each ticket's branch back into its base, then remove the worktree directory and delete the local branch. If a branch was already merged remotely (typical after a GitHub PR merge), detect that and skip straight to cleanup so we don't create a redundant merge commit.

## Phase 1: Parse Arguments

Use the same rule as `/create-worktree` so users don't have to learn two grammars. Split `$ARGUMENTS` on whitespace and classify each token:

- **Ticket number** — matches `^\d+$`, `^#\d+$`, or `^TICKET-\d+$` (case-insensitive). Normalize to a 3-digit zero-padded number.
- **Base branch** — anything else. At most one base-branch token. Two or more non-ticket tokens → report the conflict and stop.

If no ticket numbers are provided, ask the user for at least one and stop. If no base-branch token is provided, default the base to `main`.

Examples:

| Invocation | Tickets | Base |
| --- | --- | --- |
| `/merge-worktree 7` | 007 | main |
| `/merge-worktree 7 8 9` | 007, 008, 009 | main |
| `/merge-worktree 7 dev` | 007 | dev |
| `/merge-worktree TICKET-007 release-2026` | 007 | release-2026 |

## Phase 2: Resolve the Main Repo Root and Tickets

1. Confirm we're inside a git repo. Resolve the **main** repo root so the skill behaves the same whether invoked from the main checkout or from another worktree:
   ```
   MAIN_ROOT=$(dirname "$(git rev-parse --path-format=absolute --git-common-dir)")
   ```

2. For each ticket number `NNN`, glob `$MAIN_ROOT/docs/tickets/NNN-*.md`.
   - If no file matches, report `TICKET-NNN not found in docs/tickets/` and stop the whole batch — a typo in one number means the user wants to fix the input rather than half-cleanup.
   - If multiple files match, report the ambiguity and stop.

3. Extract the slug from each filename (strip the `NNN-` prefix and `.md` suffix). For each ticket, record:
   - Worktree path: `$MAIN_ROOT/.worktrees/NNN-<slug>`
   - Branch name: `ticket-NNN-<slug>`

## Phase 3: Pre-flight Checks

These run once before touching any ticket. If any of them fail, stop the entire batch — a clean refusal is much better than a half-applied state.

1. **CWD must not be inside a target worktree.** If the user's current working directory is inside any worktree we're about to remove, removing it would orphan their shell. Compare `git rev-parse --show-toplevel` against each target worktree path. If there's a match, refuse and tell the user to `cd "$MAIN_ROOT"` first.

2. **Main checkout must be clean.** The merge will land on the main checkout's HEAD, so any uncommitted changes there would be at risk. Run `git -C "$MAIN_ROOT" status --porcelain`. If it produces any output, refuse and tell the user to commit, stash, or discard.

3. **Fetch the base.** Run `git -C "$MAIN_ROOT" fetch origin <base>`. If `origin` is missing or the fetch fails (no remote, no network, no such ref), continue with the local base branch and warn the user that the result may be merging against a stale base.

4. **Resolve the base ref.** Prefer `origin/<base>`; fall back to local `<base>`. If neither exists, report `base branch '<base>' not found locally or on origin` and stop.

5. **Switch the main checkout to the base branch and fast-forward it.**
   - If local `<base>` exists: `git -C "$MAIN_ROOT" switch <base>`.
   - If local `<base>` doesn't exist (only `origin/<base>` was found): `git -C "$MAIN_ROOT" switch -c <base> origin/<base>`.
   - Then update local base to match origin: `git -C "$MAIN_ROOT" merge --ff-only origin/<base>` (skip if no `origin/<base>`). If the FF fails (local base has diverged from origin), abort and report — that's a real situation that needs human judgment, not a default decision.

## Phase 4: Per-Ticket Merge and Cleanup

Process tickets in the order given. Each ticket is independent — one ticket's failure should not stop the others.

For each `(NNN, slug, worktree_path, branch)`:

1. **Already cleaned up?** If neither `<worktree_path>` exists on disk (and is registered in `git worktree list --porcelain`) nor the branch exists (`git rev-parse --verify --quiet <branch>` fails), record as "already cleaned up" and continue.

2. **Worktree clean?** If the worktree exists, check `git -C "<worktree_path>" status --porcelain`. If non-empty, skip with a clear error (`uncommitted changes in <worktree_path>`) and continue with the next ticket. Never silently lose uncommitted work.

3. **Already merged upstream?** Run `git -C "$MAIN_ROOT" merge-base --is-ancestor <branch> <base-ref>`. If exit code is 0, the branch is already in the base — typical post-PR-merge state. Skip the merge step and go straight to step 5.

4. **Merge the branch into the base.** From the main checkout: `git -C "$MAIN_ROOT" merge --no-ff -m "Merge <branch> into <base>" <branch>`. `--no-ff` always produces a recognizable merge commit so the ticket's work can be reverted as a single unit later.
   - **On conflict**: run `git -C "$MAIN_ROOT" merge --abort` to clean up the half-merged state, record which files conflicted, skip cleanup for this ticket, and continue with the next.
   - **On other failure**: record the error, skip cleanup for this ticket, continue.

5. **Cleanup.**
   - `git -C "$MAIN_ROOT" worktree remove <worktree_path>` — removes the directory and unregisters the worktree.
   - `git -C "$MAIN_ROOT" branch -d <branch>` — safe delete. After our merge (or the upstream merge we detected) the branch is fully merged into HEAD, so `-d` will succeed.
   - If `worktree remove` fails because the worktree is missing on disk but registered, run `git -C "$MAIN_ROOT" worktree prune` and retry once.
   - If `branch -d` fails for any reason, do NOT fall back to `-D` — leave the branch and record the failure for the report. Forcing could lose unmerged work.

## Phase 5: Report

Print a summary the user can scan at a glance:

```
Merged 2 worktree(s) into dev:

  TICKET-007  merged + cleaned up
  TICKET-008  already merged upstream — cleaned up
  TICKET-009  SKIPPED — uncommitted changes in .worktrees/009-fix-search
  TICKET-010  CONFLICT — files: src/foo.ts, src/bar.ts (worktree retained)

main checkout is now on: dev (<short-hash>)
```

Group entries by outcome (merged, already-merged-and-cleaned, skipped, conflict, error) so the user immediately sees what needs follow-up.

Do not push to origin, do not delete the remote branch, and do not commit anything beyond the merge commits themselves. Push is a separate decision the user makes (often via `/commit-push-pr` or plain `git push`).
