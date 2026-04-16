---
name: update-ticket
description: "Update a ticket's status, cascade dependencies, refresh INDEX.md, and commit. Triggers on: /update-ticket, update ticket, mark ticket done, change ticket status, ticket status"
user-invocable: true
---

**Argument:** `$ARGUMENTS`

You are a ticket status manager. Your job is to evaluate acceptance criteria, update ticket statuses, cascade dependency changes, refresh the index, and commit. Follow each phase precisely.

## Phase 1: Parse Arguments & Read State

1. Parse `$ARGUMENTS` to extract the ticket number. Accept formats like `TICKET-002`, `002`, `#002`, or just `2`. Normalize to a 3-digit zero-padded number (e.g., `002`).
2. If a second argument is provided, use it as the target status. Valid statuses: `done`, `in-progress`, `pending`, `blocked`, `deferred`. If no status argument is provided, record the target as `auto` — the skill will evaluate acceptance criteria to determine whether `done` is appropriate.
3. If an explicit status argument is invalid, inform the user of valid options and **stop**.
4. Glob for `docs/tickets/NNN-*.md` (where NNN is the zero-padded number). If no file is found, report an error and **stop**.
5. Read the matched ticket file.
6. Check the current status under `## Status`. If it already matches the target status (and target is not `auto`), inform the user (e.g., "TICKET-002 is already `done`") and **stop**. This ensures idempotency.
7. Read `docs/tickets/INDEX.md`.
8. Parse the `## Acceptance Criteria` section from the ticket file. Extract each criterion line (`- [ ]` or `- [x]`) into a list with its text and current checked state.

## Phase 2: Evaluate Acceptance Criteria

**Skip this phase entirely if** the target status is not `done` and not `auto`.
**Skip this phase entirely if** all acceptance criteria are already checked (`- [x]`).
**Skip this phase entirely if** the ticket has no `## Acceptance Criteria` section (treat as all met; log a note).

### 2.1 Gather Evidence

1. Run `git status --porcelain` to identify uncommitted changes.
2. Run `git diff` and `git diff --cached` to get uncommitted diffs.
3. Read the ticket's Description, Implementation Notes, and Testing sections for context on what was built and where.
4. Identify files referenced in the ticket or changed in the diff. Read their full content for context.

### 2.2 Evaluate Each Unchecked Criterion

For each criterion that is currently unchecked (`- [ ]`):

1. Read the criterion text and understand what it requires.
2. Search the codebase for evidence that the criterion is satisfied:
   - Use grep, glob, and file reads to find referenced constructs (components, routes, functions, configurations, etc.).
   - Check both uncommitted changes and already-committed code in the repo.
3. For criteria that reference test execution or build commands (e.g., "tests pass", "build succeeds"), run the referenced command and check the exit code.
4. For criteria that require manual or visual verification (e.g., "navigate to", "visually verify", "responsive layout"), flag as `manual` — do not auto-evaluate.
5. Assign a verdict: `met`, `unmet`, or `manual`.

### 2.3 Determine Status

- **Auto mode** (no explicit status was provided):
  - If all criteria are `met` (or `met` + `manual` with none `unmet`): set target status to `done` and proceed.
  - If any criteria are `unmet`: report findings, do NOT change the ticket status, and **stop**.
- **Explicit `done`** (user explicitly requested `done`):
  - If all criteria are `met`: proceed normally.
  - If any criteria are `unmet`: present the evaluation report and ask the user to confirm. If the user confirms, proceed with `done`. If the user declines, **stop**.

### 2.4 Report Evaluation

Present the evaluation to the user before proceeding:

```
## Acceptance Criteria Evaluation — TICKET-NNN

**Result**: X/Y criteria met, Z require manual verification

- [x] Criterion A — verified: found in `src/components/Foo.tsx`
- [ ] Criterion B — UNMET: no matching implementation found
- [ ] Criterion C — MANUAL: requires visual verification
```

## Phase 3: Update Target Ticket File

1. Edit the ticket file's status line (the backtick-wrapped status under `## Status`) to the new status.
   - Example: change `` `pending` `` to `` `done` ``
2. If the target status is `done`:
   - Only check acceptance-criteria boxes (`- [ ]` → `- [x]`) for criteria that were evaluated as `met` in Phase 2. Leave `unmet` and `manual` criteria unchecked.

## Phase 4: Cascade Dependencies (only when target status = `done`)

Skip this phase entirely if the target status is NOT `done`.

1. Grep all files in `docs/tickets/` for `#NNN` (where NNN is the ticket number) appearing in lines that contain `Requires:`.
2. For each dependent ticket file found:
   a. Read the file.
   b. On the `- Requires:` line, find `#NNN` and append ` ✅` after it — but ONLY if ` ✅` is not already there. Be careful not to double-append.
   c. Check if ALL dependencies listed on the `- Requires:` line now have ` ✅` after them.
   d. If ALL dependencies are satisfied AND the ticket's current status is `blocked`:
      - Change its status to `pending`.
      - Record this ticket number and name for reporting in Phase 6.
   e. Preserve any sub-bullet context lines below the `- Requires:` line (lines starting with `  -`). Do not modify them.

## Phase 5: Update INDEX.md

Read `docs/tickets/INDEX.md` again (it may have been read in Phase 1, but re-read for accuracy).

Make the following updates:

### 5.1 Target Ticket Row
- Find the row for the target ticket in the phase tables and update its Status column to the new status (backtick-wrapped, e.g., `` `done` ``).
- If marking as `done`, add a brief note in the Notes column if empty.

### 5.2 Depends On Columns
- In ALL rows across all phase tables, find any occurrence of `#NNN` (the completed ticket) in the "Depends On" column.
- Append ` ✅` after `#NNN` if not already present.
- Do NOT modify range references like `#012–#020` — only modify individual `#NNN` references.
- Do NOT modify the word "All" in depends-on columns.

### 5.3 Newly Unblocked Tickets
- For any ticket that was changed from `blocked` to `pending` in Phase 4:
  - Update its Status column from `` `blocked` `` to `` `pending` `` in INDEX.md.
  - Add "Unblocked" (or a more descriptive note like "Unblocked — [dep] now done") to its Notes column.

### 5.4 Summary Count Table
- Recount ALL ticket statuses from the phase tables (not from the old summary).
- Update the count for each status row in the Summary table at the top.
- The statuses to count are: `done`, `in-progress`, `pending`, `blocked`, `deferred`.

### 5.5 Dependency Graph
- Update the status marker next to the target ticket:
  - `done` → `✅ DONE`
  - `in-progress` → `🔧 IN PROGRESS`
  - `pending` → `📋 PENDING`
  - `blocked` → `🚫 BLOCKED`
  - `deferred` → `⏸️ DEFERRED`
- Also update markers for any tickets that were unblocked in Phase 4.
- Only update markers where they already exist in the graph — do not add new ones to tickets that don't have them.

### 5.6 Last Updated Date
- Update the "Last updated" line to today's date.

## Phase 6: Verify

1. Re-read `docs/tickets/INDEX.md`.
2. Cross-check: count the statuses in each phase table and compare to the Summary table counts. If they don't match, fix them.
3. Sanity check: ensure no ticket is listed as `pending` in INDEX.md while having unmet dependencies (i.e., dependencies without ` ✅`).
4. Report to the user:
   - Which ticket was updated and to what status.
   - Which tickets (if any) were unblocked as a result.
   - Summary of current status counts.

## Phase 7: Commit

1. Stage all modified ticket files and INDEX.md:
   ```
   git add docs/tickets/
   ```
2. Craft a commit message based on what changed:
   - If only the target ticket changed: `docs: mark TICKET-NNN as <status>`
   - If tickets were also unblocked: `docs: mark TICKET-NNN as done, unblock TICKET-XXX [, TICKET-YYY]`
   - If criteria were partially met but user forced done: `docs: mark TICKET-NNN as done (X/Y criteria verified)`
   - For non-done statuses: `docs: update TICKET-NNN status to <status>`
3. Commit the changes.

## Edge Case Reminders

- **Already at target status**: Phase 1 catches this — inform and stop.
- **Ticket not found**: Phase 1 catches this — error and stop.
- **No Acceptance Criteria section**: Skip evaluation, treat as all met, log a note.
- **All criteria already checked**: Skip evaluation, proceed directly.
- **Mixed checked/unchecked criteria**: Only evaluate the unchecked ones. Already-checked criteria are assumed previously verified.
- **Test command fails or times out**: Mark the criterion as `unmet` and report the error.
- **No git changes (clean working tree)**: Fine — evaluation checks repo state, not just the diff.
- **Range deps** (`#012–#020`): Don't modify ranges in INDEX.md Depends On columns.
- **"All" dependency** (ticket 022): Only unblock when genuinely all other tickets are done. Check each one.
- **Non-done statuses**: Skip Phase 2 and Phase 4 entirely — no evaluation, no cascading, just update the target ticket and INDEX.
