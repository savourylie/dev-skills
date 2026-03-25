# dev-skills

Software development workflow skills for [Claude Code](https://claude.ai/claude-code).

## Skills

### `/implement`
Implements tickets from a project's ticket tracker (`docs/tickets/`), one by one. Reads the PRD and design doc, picks the next pending ticket, implements it, runs code review, commits, updates ticket status, and loops until all tickets are done.

### `/code-review`
Reviews code changes for bugs using a structured priority system (P0-P3). Supports uncommitted changes, branch diffs, PR diffs, and ticket-based review. Applies strict bug detection criteria to minimize false positives.

### `/update`
Updates a ticket's status, cascades dependency changes to dependent tickets, refreshes `INDEX.md` counts and dependency graph, and commits.

## Install

```
/plugin marketplace add savourylie/dev-skills
/plugin install dev-skills@savourylie
```

## Usage

```
/implement                  # Implement all pending tickets
/implement TICKET-003       # Implement a specific ticket

/code-review                # Auto-detect: uncommitted or branch diff
/code-review main           # Compare HEAD against main
/code-review --pr 42        # Review a pull request
/code-review 42             # Review against ticket #42

/update TICKET-003 done     # Mark a ticket as done
/update 5 in-progress       # Update ticket status
```

## Project Structure

These skills expect your project to have:
- `docs/PRD.md` — Product requirements document
- `docs/DESIGN.md` — Architecture and design document
- `docs/tickets/INDEX.md` — Ticket index with status tracking
- `docs/tickets/TICKET-NNN.md` — Individual ticket files
