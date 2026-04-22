---
name: create-tickets
description: "Generate dev tickets from requirements documents, or append new tickets to an existing project from a features catalog. MUST invoke when a user provides a PRD, product spec, or requirements doc and wants tickets created; provides a FEATURES.md / feature catalog and wants to add new features to an existing ticket set; asks to break down, split, or decompose requirements into dev work items; or references docs/tickets/ for review, audit, reordering, or fixes. Two modes: PRD mode (greenfield) and FEATURES mode (append to existing project — auto-detects existing numbering and continues it). Triggers on: /create-tickets, create tickets, break down into tasks, ticket this out, dev tickets from PRD, plan development work from requirements, add features to project, ticket new features, extend ticket tracker, review tickets in docs/tickets/"
user-invocable: true
---

**Argument:** `$ARGUMENTS`

Turn a PRD (and optional design/UX/reference documents) into a set of well-ordered, independently-completable development tickets, or extend an existing ticket set with new features from a FEATURES.md catalog.

## Inputs

Arguments are key-value pairs separated by spaces. Keys are case-insensitive.

| Argument | Required | Description |
| --- | --- | --- |
| `PRD:<file_path>` | PRD mode only (or auto-detected) | Path to the product requirements document |
| `FEATURES:<file_path>` | FEATURES mode only | Path to a feature catalog; adds new features to an existing project |
| `DESIGN:<file_path>` | No | Path to design/architecture document |
| `UX:<file_path>` | No | Path to UX specification document |
| `MISC:<path1>,<path2>` | No | Comma-separated paths to additional reference files |

### Modes

This skill has two modes, chosen by which of `PRD:` or `FEATURES:` is passed:

- **PRD mode (greenfield)** — the default. Generates a full ticket set from scratch. Used when starting a new project or when a new PRD supersedes the ticket tracker.
- **FEATURES mode (append)** — for adding new features to a project that already has tickets in `docs/tickets/`. Automatically appends without prompting; continues the existing ticket numbering pattern; reads existing tickets to avoid re-ticketing capabilities that are already built; merges new rows into the existing `INDEX.md` rather than overwriting it.

`PRD:` and `FEATURES:` are mutually exclusive. If both are passed, the skill reports the conflict and stops. `DESIGN:`, `UX:`, and `MISC:` are accepted in either mode as supplementary context.

**Examples:**
```
/create-tickets PRD:docs/PRD.md DESIGN:docs/DESIGN.md
/create-tickets PRD:docs/PRD.md UX:docs/UX.md MISC:docs/API.md,docs/MIGRATION.md
/create-tickets
/create-tickets FEATURES:docs/FEATURES.md
/create-tickets FEATURES:docs/FEATURES.md DESIGN:docs/DESIGN.md
```

## Phase 1: Gather Inputs

1. Parse `$ARGUMENTS` by splitting on whitespace. Match each token against the `PRD:`, `FEATURES:`, `DESIGN:`, `UX:`, and `MISC:` prefixes (case-insensitive). For `MISC:`, split the value on commas to get individual file paths.

2. **Mode detection**:
   - If a `FEATURES:` argument is present → **FEATURES mode**. If `PRD:` is also present, report the conflict ("FEATURES mode is for appending to existing projects; PRD mode is for greenfield — pass one or the other, not both") and **stop**.
   - Otherwise → **PRD mode**.

3. **PRD mode** — PRD discovery and existing-ticket handling:
   a. If no `PRD:` argument is provided:
      - Glob `docs/tickets/PRDv*.md` — if matches exist, pick the one with the highest version number.
      - If no versioned PRD, check for `docs/tickets/PRD.md`.
      - If nothing in `docs/tickets/`, fall back to `docs/PRD.md`.
      - If still nothing found, tell the user no PRD was found and **stop**.
   b. Create `docs/tickets/` if it does not exist.
   c. If any `NNN-*.md` ticket files already exist in `docs/tickets/`, warn the user and ask whether to:
      - **Overwrite** — delete existing tickets and start fresh. Numbering restarts at 001 with 3-digit padding.
      - **Append** — keep existing tickets. Run the numbering-pattern detection in step 5 below; new tickets start at `max_num + 1` with the existing zero-padding width.
      - **Abort** — stop without changes.

4. **FEATURES mode** — verify the existing project:
   a. Confirm the `FEATURES:` file path exists. If not, report the error and **stop**.
   b. Confirm `docs/tickets/` exists and contains at least one `NNN-*.md` ticket file. If not, report: "FEATURES mode requires an existing project with tickets in `docs/tickets/`. For a new project, use PRD mode instead." and **stop**.
   c. Confirm `docs/tickets/INDEX.md` exists — it carries the phase groupings needed for merging. If missing, report the error and **stop**.
   d. Run the numbering-pattern detection in step 5.

5. **Numbering-pattern detection** (runs in FEATURES mode always, and in PRD+Append):
   - Glob `docs/tickets/*.md` and filter to names matching `^(\d+)-.+\.md$`.
   - Let `max_num` be the largest captured integer, and `width` be the digit-length of that file's numeric prefix (e.g., `042-foo.md` → width 3; `0042-foo.md` → width 4). Preserving width matters when a project chose a non-default padding — sibling skills glob by zero-padded prefix, so mixing widths would break them.
   - New tickets begin at `max_num + 1`, zero-padded to `width` digits.
   - For checkpoint numbering, grep `docs/tickets/*.md` and `docs/tickets/INDEX.md` for `Checkpoint N` and `TEST: Checkpoint N`. The next checkpoint number is `max_found + 1` (or `0` if none found).

6. Read all input files:
   - **PRD mode**: the PRD (required); DESIGN, UX, MISC files (if provided); `CLAUDE.md` at the project root (if it exists) — for tech stack constraints, coding standards, and architectural decisions that affect how tickets are scoped.
   - **FEATURES mode**: the FEATURES file (required); every existing `docs/tickets/NNN-*.md` file (required — needed to detect which features are already built); `docs/tickets/INDEX.md` (required); DESIGN, UX, MISC files (if provided); `CLAUDE.md` at the project root (if it exists).

7. If any specified file path does not exist, report the error and **stop**.

## Phase 2: Analyze & Plan

Read all input files carefully before writing anything. In **PRD mode**, build a full plan from scratch. In **FEATURES mode**, first build a mental model of what already exists, then plan only the gap — see the FEATURES-mode additions at the end of this phase.

As you read, identify:

- **Features** — every distinct user-facing capability described in the PRD
- **Infrastructure concerns** — tech stack setup, storage, API integrations, build tooling
- **Design constraints** — component patterns, tokens, typography, color systems, animations (from DESIGN and UX docs)
- **Open questions** — anything the PRD flags as TBD or "open question" — note these as assumptions in relevant tickets rather than blocking on them
- **Out of scope** — explicitly listed out-of-scope items. Do not create tickets for these.

Build a dependency graph before writing any tickets:

1. **What must exist first?** Project scaffolding, design tokens, core data models, base components.
2. **What depends on what?** A chat UI needs the base layout. Audio input needs the chat interface. Export needs session storage.
3. **What can be parallelized?** Independent features that share no dependencies can be worked on in any order once their shared foundation is done.

Group the work into phases:
- **Phase 1 — Foundation**: Project setup, design tokens, core infrastructure
- **Phase 2 — Core Features**: The main capabilities described in the PRD
- **Phase 3 — Polish & QA**: Integration, edge cases, final QA pass

Use more phases if the project warrants it (e.g., Phase 2a and 2b if core features have a natural split). The phases are for human readability in the INDEX — they don't affect the dependency numbers.

Plan checkpoint placement — test gate tickets that verify work before proceeding:

1. **Feature checkpoints** — after each group of 2–5 implementation tickets that together deliver a testable outcome, plan a checkpoint ticket. The checkpoint tests that specific feature in isolation.
2. **Phase checkpoints** — at the end of each phase, plan a checkpoint that verifies the integration of everything built in that phase. This gates entry to the next phase. If a phase already ends with a feature checkpoint that covers the full phase, it can double as the phase checkpoint.
3. **Final checkpoint** — the very last ticket is always an end-to-end checkpoint that tests the complete system.

Checkpoint tickets are dependencies: the next group of implementation tickets should list the preceding checkpoint in their `Requires:` line. This enforces the gate.

### FEATURES-mode additions

Before writing new tickets in FEATURES mode:

1. **Build the existing-work inventory** — from the existing ticket files you read in Phase 1, extract a short list: each ticket's title plus a one-line summary of what it delivered (from its Description + Acceptance Criteria). Also note the existing phase groupings from `INDEX.md`. This is your map of what's already built.

2. **Identify gaps** — walk the FEATURES.md catalog feature by feature. For each feature, classify:
   - **Already covered** — an existing ticket (or group of tickets) clearly delivered this. Skip it and note it for the Phase 6 summary.
   - **Partially covered** — some part exists but the feature as described in FEATURES.md has unbuilt sub-capabilities. Plan tickets only for the unbuilt parts.
   - **Not covered** — no existing ticket addresses this. Plan new tickets.

3. **Dependencies on existing tickets** — new tickets may legitimately depend on existing done, in-progress, or even pending tickets. Reference them by their existing number (e.g., `Requires: #014`). Ticket numbers stay stable across modes.

4. **Phase placement** — decide whether new tickets extend the last existing phase or form a new one:
   - If the new features thematically fit the most recent phase (e.g., both are "Polish & QA" items), extend that phase.
   - If they represent a distinct new theme or a significant chunk of work, add `Phase N+1 — [theme]`.
   - Err toward a new phase when in doubt — it keeps phase tables readable.

5. **Checkpoint planning** — same rules as PRD mode, continuing the checkpoint sequence from `next_checkpoint` detected in Phase 1. Do not add a new project-wide "final end-to-end" checkpoint if one already exists; add a feature or phase checkpoint for the new batch instead.

## Phase 3: Write Ticket Files

Read this skill's `references/TEMPLATE.md` for the ticket format.

Each ticket should represent one focused session of work — roughly what a developer could complete in a day:

- **1–3 files changed** per ticket, with one clear outcome
- **Independently testable** — every ticket has a concrete "done" state you can verify
- **No bundling of unrelated concerns** — if a ticket touches both the data layer and a UI component that aren't tightly coupled, split them
- **Err on the side of smaller** — two small tickets are better than one overloaded ticket
- **First ticket** — in PRD mode, the first ticket is always project scaffolding (repo init, dependency installation, design token setup, base layout). In FEATURES mode, the first new ticket is simply the next-numbered ticket — there is no scaffolding ticket, since the project already exists.
- **Last ticket** — in PRD mode, the last ticket is always the final end-to-end checkpoint (see checkpoint rules below). In FEATURES mode, the last new ticket is typically a feature or phase checkpoint for the newly-added batch; do not create a redundant project-wide end-to-end checkpoint if the existing set already has one.

Save each ticket to `docs/tickets/` with the filename pattern `NNN-kebab-case-title.md` (e.g., `001-project-setup.md`, `002-design-tokens.md`). Use 3-digit padding by default (greenfield PRD mode and PRD+Overwrite). In FEATURES mode and PRD+Append, use the `width` detected in Phase 1 and start from `max_num + 1` — do not renumber or rewrite any existing ticket files.

Rules for each ticket:

- **Header**: Use `# [TICKET-NNN] Title` format.
- **Status**: Set to `pending` if all dependencies are met (or it has none). Set to `blocked` if it depends on unfinished tickets.
- **Dependencies**: Use `- Requires: #NNN, #NNN` format. If none, write `- Requires: None`.
- **Acceptance Criteria**: Write specific, testable statements. Not "it works" but "the chat input accepts text and sends it on Enter, displaying the message in the conversation view." Minimum 2–3 criteria per ticket.
- **Design Reference**: For UI tickets, reference specific sections from the design document (e.g., "§ Typography > Scale", "§ Components > Buttons"). Delete this section entirely for non-UI tickets.
- **Visual Reference**: For frontend tickets, describe what the user should see when the ticket is complete — specific enough that someone could visually verify it. Delete this section entirely for non-UI tickets.
- **Implementation Notes**: Key files to create/modify, architectural decisions, gotchas. Reference CLAUDE.md conventions here if applicable.
- **Testing**: How to verify the ticket is complete — commands to run, URLs to visit, expected behavior.

### Checkpoint Tickets

For each checkpoint position identified in Phase 2, write a checkpoint ticket. Read this skill's `references/TEMPLATE.md` — "Checkpoint Ticket Variant" section for the format.

- **Filename**: `NNN-test-checkpoint-N-kebab-description.md` (feature checkpoints) or `NNN-test-phaseN-checkpoint.md` (phase checkpoints)
- **Header**: `# [TICKET-NNN] TEST: Checkpoint N — What's Being Tested` or `# [TICKET-NNN] TEST: Phase N Checkpoint — Phase Summary`
- **Description**: State what tests to execute, that this is a gate, and what must pass before proceeding. Include 2–3 paragraphs: context on what was just built, what this checkpoint verifies, and what is gated by it.
- **Acceptance Criteria**: Specific pass/fail test cases — not code changes. Each criterion is a concrete verification step.
- **Implementation Notes**: Begin with "This is a manual test execution ticket — no code changes unless bugs are found during testing." Then list common failure modes, test commands, and environment notes.
- **Dependencies**: The last implementation ticket(s) in the group being tested.

Number checkpoints sequentially across the entire project (Checkpoint 0, Checkpoint 1, ...) — do not restart numbering per phase. The final ticket is always the last checkpoint. In FEATURES mode, continue the checkpoint sequence from the `next_checkpoint` detected in Phase 1.

## Phase 4: Write INDEX.md

Read this skill's `references/INDEX.md` for the index format.

### PRD mode (or PRD+Overwrite)

Generate `docs/tickets/INDEX.md` from scratch containing:

1. **Last updated date** — today's date
2. **Summary table** — counts of tickets by status, using emoji markers (✅ Done, 🔧 In Progress, 📋 Pending, 🚫 Blocked, ⏸️ Deferred)
3. **Phase tables** — one table per phase, each ticket showing: number, linked title (relative path), status (backtick-wrapped), dependencies, notes
4. **Checkpoint rows** — format checkpoint ticket links in bold: `[**TEST: Checkpoint N — Title**](./NNN-test-...)`. In the Notes column, write `Gate: Phase N` or `Gate: Final`
5. **Status key** — definition of each status value

All tickets start as either `pending` (no dependencies or all dependencies met) or `blocked` (has unmet dependencies). The summary counts should reflect the initial state.

### FEATURES mode (or PRD+Append)

Do **not** overwrite `INDEX.md`. Merge new rows into the existing file:

1. **Preserve everything existing** — all existing phase tables, rows, statuses, and notes stay untouched. Only add new rows and/or new phase sections for the newly-created tickets.
2. **Extend the last phase** — if the new tickets thematically fit the most recent phase, append their rows at the bottom of that phase's table. Checkpoint rows are bold with `Gate: Phase N` in Notes.
3. **Or add a new phase section** — insert a new `## Phase N — [theme]` heading and table below the last existing phase, above the Status Key.
4. **Update the Summary table** — recount all rows across all phase tables (existing + new) and rewrite the counts. Existing statuses are not changed by this run.
5. **Update the "Last updated" date** to today. Optionally include a short context note in parentheses (e.g., `(added TICKET-015 through TICKET-021)`) matching the style of previous updates.
6. **Do not touch** the Status Key or any structural commentary.

## Phase 5: Self-Review

This phase is mandatory. After writing all tickets, review every ticket in `docs/tickets/` for:

1. **Dependency ordering issues**
   - Can ticket N actually be started given its listed dependencies?
   - Are there circular dependencies?
   - Does any ticket depend on a ticket that doesn't exist?

2. **Missing acceptance criteria**
   - Every ticket needs at least 2–3 specific, testable criteria.
   - Criteria must be concrete (not "works correctly" or "is properly styled").

3. **Scope creep**
   - Does any ticket touch more than 3 files?
   - Does any ticket have more than 5 acceptance criteria?
   - If so, consider splitting it into smaller tickets.

4. **Gaps**
   - Is there a feature in the PRD that no ticket covers?
   - Is there infrastructure assumed but never set up?

5. **Checkpoint coverage**
   - Does every phase have at least one checkpoint?
   - Is the final ticket a checkpoint (not an implementation ticket)?
   - Do implementation tickets after a checkpoint list that checkpoint in their dependencies?
   - Are checkpoint acceptance criteria testable pass/fail statements (not code changes)?

6. **Consistency**
   - Do all tickets follow the template format?
   - Are status values correct given dependencies?
   - Does INDEX.md accurately reflect all ticket files?
   - Are checkpoint rows bold in INDEX.md with `Gate:` notes?

7. **Append-mode checks** (applies to FEATURES mode and PRD+Append; skip in greenfield / PRD+Overwrite)
   - **Numbering**: the lowest new ticket number equals `max_num + 1` from Phase 1, and all new tickets share the same zero-padding width as existing ones.
   - **Existing files untouched**: no existing ticket file was renumbered, renamed, or modified.
   - **INDEX merged, not rewritten**: all existing rows and statuses are intact; only new rows or new phase sections were added.
   - **Dependencies**: new-ticket `Requires:` lines correctly reference existing ticket numbers where dependencies are real.
   - **No duplicate work** (FEATURES mode only): for each new ticket, confirm no existing ticket already delivers the same capability. If overlap is found, either narrow the new ticket's scope or delete it.

Fix any problems you find. Update both the ticket files and INDEX.md if changes are made.

## Phase 6: Summary

Tell the user:
- Which mode ran (PRD greenfield, PRD append, or FEATURES append)
- How many tickets were created in this run (implementation + checkpoint); in FEATURES mode and PRD+Append, state the new number range (e.g., "added TICKET-015 through TICKET-021")
- How they're grouped by phase, including where checkpoints gate progress
- In FEATURES mode: which FEATURES.md entries were skipped because existing tickets already cover them
- Any assumptions made due to open questions in the source document
- The path to `docs/tickets/INDEX.md` as the project tracker
- Suggest next steps: `/implement-ticket 001` for greenfield, or `/implement-ticket <first new number>` to start the new batch
