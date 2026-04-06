---
name: create-tickets
description: "Generate dev tickets from requirements documents. MUST invoke when a user provides a PRD, product spec, or requirements doc and wants tickets created; asks to break down, split, or decompose requirements into dev work items; or references docs/tickets/ for review, audit, reordering, or fixes. Triggers on: /create-tickets, create tickets, break down into tasks, ticket this out, dev tickets from PRD, plan development work from requirements, review tickets in docs/tickets/"
user-invocable: true
---

**Argument:** `$ARGUMENTS`

Turn a PRD (and optional design/UX/reference documents) into a set of well-ordered, independently-completable development tickets.

## Inputs

Arguments are key-value pairs separated by spaces. Keys are case-insensitive.

| Argument | Required | Description |
| --- | --- | --- |
| `PRD:<file_path>` | Yes (or auto-detected) | Path to the product requirements document |
| `DESIGN:<file_path>` | No | Path to design/architecture document |
| `UX:<file_path>` | No | Path to UX specification document |
| `MISC:<path1>,<path2>` | No | Comma-separated paths to additional reference files |

**Examples:**
```
/create-tickets PRD:docs/PRD.md DESIGN:docs/DESIGN.md
/create-tickets PRD:docs/PRD.md UX:docs/UX.md MISC:docs/API.md,docs/MIGRATION.md
/create-tickets
```

## Phase 1: Gather Inputs

1. Parse `$ARGUMENTS` by splitting on whitespace. Match each token against the `PRD:`, `DESIGN:`, `UX:`, and `MISC:` prefixes (case-insensitive). For `MISC:`, split the value on commas to get individual file paths.

2. **PRD auto-detection** — if no `PRD:` argument is provided:
   a. Glob `docs/tickets/PRDv*.md` — if matches exist, pick the one with the highest version number.
   b. If no versioned PRD, check for `docs/tickets/PRD.md`.
   c. If nothing in `docs/tickets/`, fall back to `docs/PRD.md`.
   d. If still nothing found, tell the user no PRD was found and **stop**.

3. Create `docs/tickets/` if it does not exist.

4. If any `NNN-*.md` ticket files already exist in `docs/tickets/`, warn the user and ask whether to:
   - **Overwrite** — delete existing tickets and start fresh
   - **Append** — start numbering from the next available number
   - **Abort** — stop without changes

5. Read all input files:
   - The PRD (required)
   - DESIGN, UX, MISC files (if provided)
   - `CLAUDE.md` at the project root (if it exists) — for tech stack constraints, coding standards, and architectural decisions that affect how tickets are scoped

6. If any specified file path does not exist, report the error and **stop**.

## Phase 2: Analyze & Plan

Read all input files carefully before writing anything. As you read, identify:

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

## Phase 3: Write Ticket Files

Read this skill's `references/TEMPLATE.md` for the ticket format.

Each ticket should represent one focused session of work — roughly what a developer could complete in a day:

- **1–3 files changed** per ticket, with one clear outcome
- **Independently testable** — every ticket has a concrete "done" state you can verify
- **No bundling of unrelated concerns** — if a ticket touches both the data layer and a UI component that aren't tightly coupled, split them
- **Err on the side of smaller** — two small tickets are better than one overloaded ticket
- **First ticket is always project scaffolding** — repo init, dependency installation, design token setup, base layout
- **Last ticket is always the final end-to-end checkpoint** — see checkpoint rules below

Save each ticket to `docs/tickets/` with the filename pattern `NNN-kebab-case-title.md` (e.g., `001-project-setup.md`, `002-design-tokens.md`).

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

Number checkpoints sequentially across the entire project (Checkpoint 0, Checkpoint 1, ...) — do not restart numbering per phase. The final ticket is always the last checkpoint.

## Phase 4: Write INDEX.md

Read this skill's `references/INDEX.md` for the index format.

Generate `docs/tickets/INDEX.md` containing:

1. **Last updated date** — today's date
2. **Summary table** — counts of tickets by status, using emoji markers (✅ Done, 🔧 In Progress, 📋 Pending, 🚫 Blocked, ⏸️ Deferred)
3. **Phase tables** — one table per phase, each ticket showing: number, linked title (relative path), status (backtick-wrapped), dependencies, notes
4. **Checkpoint rows** — format checkpoint ticket links in bold: `[**TEST: Checkpoint N — Title**](./NNN-test-...)`. In the Notes column, write `Gate: Phase N` or `Gate: Final`
5. **Status key** — definition of each status value

All tickets start as either `pending` (no dependencies or all dependencies met) or `blocked` (has unmet dependencies). The summary counts should reflect the initial state.

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

Fix any problems you find. Update both the ticket files and INDEX.md if changes are made.

## Phase 6: Summary

Tell the user:
- How many tickets were created (implementation + checkpoint)
- How they're grouped by phase, including where checkpoints gate progress
- Any assumptions made due to open questions in the PRD
- The path to `docs/tickets/INDEX.md` as the project tracker
- Suggest next steps: `/implement-ticket 001` to start implementing
