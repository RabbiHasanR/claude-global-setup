---
name: docs
description: Generate or update project documentation — README, API docs, inline docs, changelog, contributing guide, setup guide, architecture doc, howto guides, runbook, ADR, environment docs. Use when writing or improving any project documentation.
---

Handle documentation task based on $ARGUMENTS:

**Default output:** `docs/` folder unless user specifies otherwise. Create `docs/` if it doesn't exist.

**Preamble:** Include sections only if applicable to the detected project. Skip sections that have no relevant content.

**Config files to scan** (reuse across all workflows): `package.json`, `pyproject.toml`, `setup.cfg`, `requirements.txt`, `Dockerfile`, `docker-compose*.yml`, `.env.example`, `.github/workflows/*.yml`

## Forking Rules
**Fork to subagent** (scans many files): `readme`, `api`, `architecture`, `setup`, `contributing`, `howto`, `runbook`, `env`
**Run in main context** (lightweight, user-driven): `inline`, `changelog`, `adr`

When forking: scan the project, generate the full document, return it. Present result for user review before saving.

---

## readme (fork)
1. Scan config files and project structure
2. Check for existing `LICENSE` file and `CONTRIBUTING.md`
3. Generate or update `README.md` at project root:
   - Project title and one-line description
   - ## Prerequisites — required tools/versions
   - ## Setup — how to get running locally
   - ## Usage — key commands (dev, build, test, lint)
   - ## Project Structure — brief directory overview
   - ## Environment Variables — list required vars (never actual values)
   - ## Contributing — link to `CONTRIBUTING.md` if it exists
   - ## License — detected from LICENSE file, or omit if none

---

## api (fork)
1. Detect API type by scanning for:
   - **FastAPI/Django DRF**: `@router.get/post/put/delete`, `@app.get/post`, `ViewSet`, `APIView`, `urls.py` with `path()`/`re_path()`
   - **Express/Fastify**: `router.get/post`, `app.get/post`
   - **Django**: `path()`, `re_path()` in `urls.py`
2. For each endpoint: method, path, request body/params, response shape, auth required, status codes
3. Group by resource/module
4. Output as structured markdown in `docs/api.md`

---

## architecture (fork)
1. Read project structure, entry points, config, key modules
2. Generate `docs/architecture.md`:
   - System overview — what it does and who uses it
   - High-level components and how they connect
   - Directory structure with purpose of each
   - Data flow — how a request travels through the system
   - Key design decisions and why
   - External services/APIs and databases it depends on

---

## setup (fork)
1. Scan config files, Docker setup, env vars, lock files (`poetry.lock`, `package-lock.json`, `yarn.lock`)
2. Generate `docs/setup.md`:
   - System requirements (OS, language versions, tools)
   - Installation from scratch (reference lock files for exact dep versions)
   - Environment variable setup (list all, explain each)
   - Database setup
   - How to run locally (with and without Docker if both exist)
   - Common setup issues and solutions
   - How to verify everything works

---

## contributing (fork)
1. Scan for: linter config, CI pipeline (`.github/workflows/*.yml`), branch strategy
2. Detect test runner:
   - Python: `pytest` or `unittest` from `pyproject.toml`/`setup.cfg`
   - JS/TS: `jest`, `vitest`, `mocha` from `package.json`
3. Generate `CONTRIBUTING.md`:
   - Dev environment setup
   - Branch naming convention (`feat/`, `fix/`, `refactor/`, `chore/`, `docs/`)
   - Commit message format (conventional commits)
   - How to run tests and linting (with detected commands)
   - PR process and review expectations
   - Code style guidelines detected from config

---

## howto \<feature\> (fork)
1. Read relevant code files for the specified feature/process
2. Generate `docs/howto-<name>.md`:
   - What it does and why it exists
   - How it works (trace the code flow)
   - Key files and functions involved
   - How to modify or extend it
   - Gotchas or edge cases

---

## runbook (fork)
Ops/incident guide for production systems.
1. Scan: `docker-compose*.yml`, Kubernetes manifests (`k8s/`, `deploy/`, `manifests/`), `.github/workflows/*.yml`, `Dockerfile`, `.env.example`
2. Generate `docs/runbook.md`:
   - Service overview — what runs, on what infra
   - Health checks — endpoints and expected responses
   - How to deploy — step-by-step for each environment
   - How to rollback — procedure and commands
   - Common alerts — what they mean and what to do
   - Logs — where they are and how to query (CloudWatch, Grafana, kubectl logs)
   - Escalation path — who to contact and when

---

## env (fork)
Environment variable and deployment reference.
1. Scan: `.env.example`, `docker-compose*.yml`, `.github/workflows/*.yml`, Kubernetes ConfigMaps/Secrets under `k8s/`
2. Generate `docs/env.md`:
   - Full variable list: name, required/optional, default, description
   - Per-environment differences (local vs CI vs staging vs production)
   - How to configure locally, in CI, and in production

---

## inline (main context)
1. Scan 2–3 existing docstrings in the file to detect style:
   - Python → Google-style docstrings
   - JS/TS → JSDoc (`/** */`)
   - Match whatever style already exists
2. Read the specified file(s)
3. Add or improve docstrings/comments on public functions and classes
4. Focus on *why* and *edge cases*, not obvious *what*
5. Don't touch private/internal functions unless complex
6. Present changes for user review before saving

---

## changelog (main context)
1. Run `git describe --tags --abbrev=0 2>/dev/null` to find the last tag
   - If no tags found, use all commits
2. Run `git log <last-tag>..HEAD --oneline` to get commits since last release
3. Prompt user for version number (e.g. `1.2.0`)
4. Group commits by conventional commit type (feat, fix, refactor, perf, etc.)
5. Generate entry in Keep a Changelog format (`Added`, `Changed`, `Fixed`, `Removed`)
6. Prepend to `CHANGELOG.md` with header `## [<version>] - <YYYY-MM-DD>`

---

## adr \<title\> (main context)
Architecture Decision Record — lightweight log of a significant decision.
1. Count existing ADRs in `docs/adr/` to determine next number (`NNN`)
2. Create `docs/adr/` if it doesn't exist
3. Create `docs/adr/NNN-<slugified-title>.md` with this template:
   ```
   # NNN. <Title>

   Date: YYYY-MM-DD
   Status: Proposed

   ## Context
   [What situation led to this decision?]

   ## Decision
   [What was decided?]

   ## Consequences
   [What are the results — positive and negative?]

   ## Alternatives Considered
   [Other options evaluated and why they were rejected]
   ```
4. Fill in title, date, and any context already known from the argument
5. Present for user to complete
