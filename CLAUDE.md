# Global Preferences

## Who I Am
Software engineer. Backend-specialized (Python), DevOps experience, frontend-capable (JavaScript).

## Stack
- Languages: Python (primary), JavaScript/TypeScript (frontend), Bash
- Backend: FastAPI, Django, Django REST Framework
- Frontend: Ember.js, React (when needed)
- Database: PostgreSQL (including psql functions/procedures), MongoDB, Redis
- Infra: Docker, docker-compose, AWS (EC2, S3, RDS, CloudWatch), Kubernetes
- Observability: Grafana, Prometheus, OpenTelemetry
- CI/CD: GitHub Actions

## Code Style
- **DRY first** — never duplicate logic; extract and reuse
- Type hints on all function signatures
- Docstrings on public functions (Google style)
- snake_case everywhere, UPPER_CASE for constants
- Keep functions minimal; one function = one responsibility

## How I Work — IMPORTANT
- **NEVER jump to code.** Always: Plan → Discuss → Agree → Implement
- For any task: first explain the approach, trade-offs, and alternatives
- Wait for my approval before writing/changing any code
- If a task has multiple approaches, list them with pros/cons
- Break big tasks into numbered steps; confirm each phase before moving on
- Ask clarifying questions upfront rather than assuming

## Effort recommendation — IMPORTANT

Before responding to ANY prompt (no exceptions — including questions,
conversational replies, mid-task continuations), do this in order:

1. Look at the prompt: task type, files/services it touches, reasoning needed.
2. Output as the FIRST line:
       `RECOMMEND: <level> — <one-sentence reason citing files/scope>`
   Use the decision rules in `skills/effort-pick/SKILL.md`.
3. Then output:
       `Switch effort? Reply with a level name (low/medium/high/xhigh/max) or 'go'.`
4. STOP. Do not do the task, answer the question, or say anything else
   until I reply.

This overrides the SKIP list in `skills/effort-pick/SKILL.md` — fire on
every prompt, no exceptions.

## Workflow Rules
- Never hardcode secrets — use env vars
- Git: conventional commits (feat/fix/refactor/style/perf/test/chore/docs/ci)

## Communication
- Explain *why*, not just *what* — I'm learning
- Be direct, skip filler
- If I'm doing something wrong, tell me the better way
- Bengali or English both fine