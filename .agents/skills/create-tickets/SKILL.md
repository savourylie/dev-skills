---
name: create-tickets
description: "Generate dev tickets from requirements documents. Use when the user provides a PRD, product spec, or requirements doc and wants tickets created, or asks to break down requirements into development work items. Prefer explicit invocation with $create-tickets."
---

**Argument:** `$ARGUMENTS`

Turn a PRD and optional design/UX/reference documents into well-ordered development tickets.

## Inputs

Arguments are key-value pairs separated by spaces:

- `PRD:<file_path>` — path to the product requirements document (auto-detected if omitted)
- `DESIGN:<file_path>` — optional design/architecture document
- `UX:<file_path>` — optional UX specification
- `MISC:<path1>,<path2>` — optional comma-separated reference files

## Phase 1: Gather Inputs

1. Parse `$ARGUMENTS` by matching tokens against `PRD:`, `DESIGN:`, `UX:`, `MISC:` prefixes. For `MISC:`, split on commas.

2. If no `PRD:` given, auto-detect:
   a. Glob `docs/tickets/PRDv*.md` → highest version number
   b. Fall back to `docs/tickets/PRD.md`
   c. Fall back to `docs/PRD.md`
   d. If nothing found → error and stop

3. Create `docs/tickets/` if it does not exist.

4. If `NNN-*.md` ticket files already exist in `docs/tickets/`, ask the user: overwrite, append (start from next number), or abort.

5. Read all input files plus `CLAUDE.md` at project root if present.

## Phase 2: Analyze & Plan

Read all input files. Identify features, infrastructure concerns, design constraints, open questions (note as assumptions), and out-of-scope items.

Build a dependency graph:
1. What must exist first? (scaffolding, tokens, data models)
2. What depends on what?
3. What can be parallelized?

Group into phases: Foundation → Core Features → Polish & QA. Add more phases if warranted.

## Phase 3: Write Ticket Files

Read `./references/TEMPLATE.md` for the format.

Each ticket = one focused day of work. File naming: `NNN-kebab-case-title.md` in `docs/tickets/`.

Rules:
- Header: `# [TICKET-NNN] Title`
- Status: `pending` (deps met) or `blocked` (deps unmet)
- Dependencies: `- Requires: #NNN, #NNN` or `- Requires: None`
- Acceptance criteria: 2–3 minimum, specific and testable
- Design/Visual Reference: include for UI tickets, delete for non-UI
- Implementation Notes: key files, decisions, gotchas
- Testing: how to verify completion

## Phase 4: Write INDEX.md

Read `./references/INDEX.md` for the format.

Generate `docs/tickets/INDEX.md` with:
- Today's date
- Summary table with emoji status markers (✅ Done, 🔧 In Progress, 📋 Pending, 🚫 Blocked, ⏸️ Deferred)
- Phase tables: number, linked title, backtick-wrapped status, dependencies, notes
- Status key

## Phase 5: Self-Review

Review ALL tickets for:
1. Dependency ordering issues (circular deps, missing deps)
2. Missing acceptance criteria (<2 or vague)
3. Scope creep (>3 files or >5 criteria → split)
4. Gaps (PRD features with no ticket, missing infrastructure)
5. Consistency (template format, status correctness, INDEX accuracy)

Fix any problems found. Update both ticket files and INDEX.md.

## Phase 6: Summary

Report: ticket count, phase grouping, assumptions made, path to INDEX.md. Suggest `/implement-ticket 001` to start.
