---
name: implement-ticket
description: "Implement a specific ticket from the ticket tracker with code review. Triggers on: /implement-ticket TICKET-NNN, implement TICKET-NNN, implement ticket NNN, implement ticket 3"
user-invocable: true
---

**Argument:** `$ARGUMENTS`

You are a senior full-stack developer implementing a ticket. Follow this workflow precisely. Do not skip steps.

A ticket ID is required (e.g., `TICKET-001`, `001`, or `1`). If no argument is provided, inform the user that a ticket ID is required and stop.

## Phase 1: Understand the Project

1. Read `docs/PRD.md` thoroughly. Internalize the product requirements, user stories, acceptance criteria, and scope.
2. Read `docs/DESIGN.md` thoroughly. Understand the architecture, data models, API contracts, tech stack choices, and any design decisions or constraints.
3. Briefly summarize (to yourself) the key requirements and architectural decisions before moving on. This is your mental model for all implementation work.

## Phase 2: Load the Ticket

1. Read `docs/tickets/INDEX.md` to see the current status of all tickets and understand dependencies.
2. Select the ticket specified in the argument. If it is already marked as done, inform the user and stop.
3. Read the full ticket file (e.g., `docs/tickets/TICKET-001.md`) for the selected ticket.
4. Before writing any code, briefly state:
   - What you're implementing
   - Which files you expect to create or modify
   - Any edge cases or risks you see

## Phase 3: Implement the Ticket

1. Implement the ticket fully, following the design in `DESIGN.md` and the requirements in the ticket.
2. Write clean, well-structured code. Follow existing project conventions (naming, file structure, patterns).
3. Include appropriate error handling, input validation, and edge case coverage.
4. If the ticket specifies tests, write them. If it doesn't but the project has a test suite, add tests for your changes anyway.
5. Make sure any new files are properly exported/imported and integrated with the rest of the codebase.
6. Prepare manual testing instructions for the implementation. Think about what a developer or QA tester would need to do to verify the feature works correctly by hand — specific commands to run, URLs to visit, inputs to provide, expected outputs to observe, and edge cases to try.

## Phase 4: Code Review

### 4a: Build Check

Run lint, type-check, and build commands from `package.json` (e.g., `npm run lint`, `npm run build`). Fix any errors until the build is clean.

### 4b: Automated Code Review

This step is mandatory — do not skip it. Invoke the `/review-ticket` skill using the `Skill` tool to review all uncommitted changes against the ticket requirements:

```
skill: "review-ticket"
```

### 4c: Fix and Re-review

If the code review finds any issues:
1. Fix them immediately.
2. Re-run the build check (4a) until clean.
3. Re-invoke `/review-ticket` (4b) to verify fixes.
4. Repeat until both build and code review are clean.

Do not proceed to the next phase until the build passes cleanly AND the code review returns no P0 or P1 findings.

## Phase 5: Summary and Manual Testing

Present the following to the user:

### Implementation Summary
1. State which ticket was implemented (ID and title).
2. Briefly summarize what was built — key files created or modified, architectural decisions made.
3. Note any deviations from the ticket spec or design doc, and why.
4. Note any remaining concerns, tech debt, or follow-up items.

### Manual Testing Instructions
Provide clear, step-by-step instructions for how to manually verify the implementation works correctly. Include:
- Prerequisites (environment setup, dependencies, services that need to be running)
- Exact commands to run the application or relevant part of it
- Specific actions to take (URLs to visit, buttons to click, inputs to provide)
- Expected results for each action
- Edge cases worth testing manually
- If the ticket involves API changes, include example curl commands or request/response pairs

If the implementation is purely internal (e.g., a refactor with no user-facing changes), state that manual testing is not applicable and explain what the automated tests cover instead.

**Do NOT commit changes, update ticket status, or invoke the `/update-ticket`, `/commit-ticket`, or `/commit-push-pr` skills.** The user will handle those steps separately.
