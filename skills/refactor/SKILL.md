---
name: refactor
description: Refactor code — improve structure, readability, DRY, naming, type annotations, error handling. Use when cleaning up, restructuring, or improving code quality.
---

Fork for multi-file/module targets. Main context for single file or function.

---

## Process

1. **Read & understand** — what does this code do, what patterns does it follow
2. **Identify issues** — check for:
   - DRY violations
   - Functions with multiple responsibilities
   - Deep nesting (3+ levels)
   - Unclear naming (`data`, `result`, `temp`, `info`, `handle`, `process` — flag these)
   - Missing type annotations (Python: all function signatures)
   - Missing error handling
   - Tight coupling
   - Dead code, unused imports
   - Time/space complexity — inefficient algorithms, unnecessary iterations, excessive memory usage
   - Code that belongs in a different file/module
   - File/folder structure not aligned with domain (mixing concerns, flat when should be grouped)
3. **Present suggestions** — for each issue: Problem → why it matters → before/after code → Priority: critical / important / nice-to-have
4. **Wait for approval** — do NOT rewrite until user confirms which suggestions to apply
5. **Apply** — implement only approved changes
6. **Verify** — detect and run test command:
   - Python: check `pyproject.toml`/`setup.cfg` for pytest config, run `pytest`
   - JS/TS: check `package.json` scripts, run `npm test` or `yarn test`
   - Makefile: check for `test` target, run `make test`
   - If no tests found, note it and suggest adding them

**Rules:** Most likely cause first. Respect existing project patterns. Don't refactor for style alone. If code is already clean, say so. Explain *why* behind each suggestion.

---

## restructure

When moving code between files:

1. Show the full plan: what moves where and why
2. List all files that import the moved code (grep for references)
3. Update all import paths after moving
4. Wait for approval before any file changes
