---
name: effort-pick
description: Decision rules for picking an /effort level. The "Effort
  recommendation" section in CLAUDE.md enforces *when* to use these rules
  (every prompt, no exceptions). This file is the *what* — how to pick a
  level for a given task.
---

# Effort level decision rules

The `CLAUDE.md` "Effort recommendation" section enforces *when* to use these
rules (every prompt). This file is the *what* — how to pick a level.

## Decision rules

Pick `low` when:
- Mechanical edits: rename, format, lint fix, typo, import cleanup
- Boilerplate matching an existing pattern in the repo
- Single-file changes with no cross-file reasoning
- Running a known command and reporting output

Pick `medium` when:
- New endpoint, model, migration, or test following established patterns
- Routine refactor within one module
- Writing docs, SKILL.md, subagent config, or README sections
- Small bug fix with a clear repro

Pick `high` when:
- Multi-file changes with real coupling between files
- Debugging across services (Django ↔ FastAPI ↔ Redis ↔ Postgres)
- PR review on architectural or security-relevant changes
- Postgres query tuning, index design, EXPLAIN ANALYZE work
- CI/CD pipeline changes with deploy consequences

Pick `xhigh` when:
- Multi-step agentic work: plan → edit → test → iterate
- Whole-feature build spanning backend + frontend
- Cross-cutting changes that need to hold a lot of context at once

Pick `max` when:
- Deep architecture design with many tradeoffs
- Gnarly performance or correctness root-cause
- Security audit on critical paths (auth, RLS, secrets, deploy promotion)
- Final review before merging risky infra changes
- Schema migrations on production tables

## Anti-patterns

- Don't recommend `max` for anything that fits a known pattern
- Don't recommend `low` if the task touches more than 2 files
- If unsure between two levels, pick the lower one
- Never lecture about effort levels — one line, then move on

## Project overrides

If the project's `CLAUDE.md` contains an "Effort routing notes" section,
those rules override the defaults above.