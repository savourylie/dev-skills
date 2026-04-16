---
name: "update-ticket"
description: "Use when the user explicitly asks to update a ticket status, cascade dependency markers, refresh docs/tickets/INDEX.md, and commit the ticket-document changes. Prefer explicit invocation with $update-ticket."
---

# Update Ticket Status

Update a ticket's status, cascade dependency state where needed, refresh `docs/tickets/INDEX.md`, and commit the documentation changes.

Parse any text that follows the skill invocation as arguments. Accept ticket identifiers like `TICKET-002`, `002`, `#002`, or `2`. Accept statuses `done`, `in-progress`, `pending`, `blocked`, and `deferred`. If no status is provided, auto-evaluate acceptance criteria to decide between `done` and keeping the current status.

## Phase 1: Read State

1. Normalize the requested ticket number to a zero-padded 3-digit identifier.
2. Validate the target status. If no status provided, record as `auto`.
3. Find the matching ticket file under `docs/tickets/`.
4. Read the ticket file and `docs/tickets/INDEX.md`.
5. If the ticket already has the requested status, report that and stop.
6. Parse the `## Acceptance Criteria` section — extract each criterion with its checked state.

## Phase 2: Evaluate Acceptance Criteria

Skip if target status is not `done` and not `auto`. Skip if all criteria are already checked or if there is no Acceptance Criteria section.

1. Gather evidence: run `git status --porcelain`, `git diff`, `git diff --cached`. Read the ticket's Description, Implementation Notes, and Testing sections. Read files referenced in the ticket or changed in the diff.
2. Evaluate each unchecked criterion against the codebase:
   - Search for evidence using grep, glob, and file reads (both uncommitted changes and committed code).
   - For criteria referencing test/build commands, run the command and check the result.
   - For criteria requiring manual/visual verification, flag as `manual`.
   - Verdict per criterion: `met`, `unmet`, or `manual`.
3. Determine status:
   - **Auto mode**: all met (or met + manual) → `done`. Any unmet → report and **stop**.
   - **Explicit `done`**: any unmet → ask user to confirm before proceeding.
4. Report the per-criterion evaluation before proceeding.

## Phase 3: Update the Ticket File

1. Update the status line under `## Status`.
2. If the target status is `done`, only check acceptance-criteria boxes for criteria evaluated as `met`. Leave `unmet` and `manual` criteria unchecked.

## Phase 4: Cascade Dependencies for `done`

If the target status is `done`:

1. Find every ticket that references the completed ticket in a `Requires:` line.
2. Mark that dependency satisfied if it is not already marked.
3. If a dependent ticket now has all dependencies satisfied and is currently `blocked`, move it to `pending`.
4. Record every ticket that became unblocked so you can update the index and report it clearly.

## Phase 5: Refresh `INDEX.md`

Update the index in place:

1. Update the target ticket row status.
2. Mark satisfied single-ticket dependencies in "Depends On" columns.
3. Update any newly unblocked ticket rows.
4. Recount the summary table totals.
5. Refresh existing dependency graph markers already present in the file.
6. Update the "Last updated" date.

Preserve range dependencies such as `#012-#020` and preserve the word `All` where the file uses it.

## Phase 6: Verify

1. Re-read the index.
2. Check that the summary counts match the phase tables.
3. Check that no ticket is marked `pending` while still showing unmet dependencies.
4. Report the updated ticket, the new status, any newly unblocked tickets, and the final status counts.

## Phase 7: Commit

1. Stage only the ticket files and `docs/tickets/INDEX.md`.
2. Create one docs-focused commit message that reflects the status change. If criteria were partially met but user forced done, note in the message: `docs: mark TICKET-NNN as done (X/Y criteria verified)`.

## Safety Rules

- If the ticket file or index is missing, stop and report the missing input.
- Do not modify unrelated docs.
- If the requested status is invalid, report the valid options and stop.
- If criteria evaluation cannot determine status, ask the user rather than guessing.
