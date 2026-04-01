---
name: debug
description: Debug errors, stack traces, test failures, build failures, infra issues, performance problems, and log analysis. Use when user shares an error, unexpected behavior, failing output, or log file.
---

Handle debug task based on $ARGUMENTS. Fork only for `logs` (heavy file scanning). Everything else runs in main context.

---

## error (default)
Use when user shares an error message, exception, or unexpected behavior.

1. **Read the file** at the reported path/line before diagnosing — don't rely on pasted snippets alone
2. **Check recent changes** — `git log --oneline -5` — most bugs come from recent commits
3. **Identify** — error type, source language/tool, file + line number
4. **Reproducibility** — consistent or intermittent? Ask if unclear — changes the diagnosis entirely
5. **Root cause** — trace to origin; for stack traces read bottom-up, skip library internals
6. **Explain** — what broke, why, where (concise)
7. **Solutions** — 2–3 approaches, each with: what it does, pros/cons, recommendation
8. **Wait for approval** — do NOT apply any fix until user picks
9. **Implement** — apply only the approved solution

Rules: most likely cause first. Ask for context rather than guessing. Show before/after. Mention prevention only for repeatable patterns.

---

## test \<output\>
Use when pytest, jest, vitest, or other test runner output is shared.

1. Identify failing test(s) — name, file, line
2. Read the test and the code under test
3. Distinguish failure type:
   - **Assertion failure** — wrong value returned; check logic
   - **Exception** — code crashed before assertion; trace the error
   - **Fixture/setup failure** — test couldn't initialize; check fixtures, mocks, DB state
4. Root cause — explain why the test fails
5. Solutions — fix the code or fix the test (clarify which is wrong)
6. Wait for approval, then implement

---

## build \<output\>
Use for Docker build, pip install, npm install, GitHub Actions, or CI failures.

1. Identify the failing step — line number in build output
2. Distinguish failure type:
   - **Dependency error** — missing package, version conflict, bad lock file
   - **Syntax/import error** — caught at build time, not runtime
   - **Config error** — bad Dockerfile, workflow YAML, or env var
   - **Network/registry error** — transient, retry or mirror issue
3. Root cause + fix
4. Wait for approval, then implement

---

## infra \<context\>
Use for Docker container, Kubernetes pod, or AWS service failures.

1. Run relevant diagnostic:
   - Docker: `docker ps -a`, `docker logs <container>`, `docker inspect <container>`
   - K8s: `kubectl describe pod <pod>`, `kubectl logs <pod>`, `kubectl get events`
2. Identify failure pattern:
   - **OOMKilled** — memory limit hit; check limits and actual usage
   - **CrashLoopBackOff** — app crashing on start; check logs for exception
   - **ImagePullBackOff** — registry auth or image name issue
   - **Pending** — scheduling failure; check node resources and taints
   - **Exit code 1/137/143** — app crash / OOM / SIGTERM respectively
3. Root cause + fix
4. Wait for approval, then implement

---

## perf \<context\>
Use for slow queries, high memory/CPU, latency spikes, timeouts.

1. **Measure first** — do not optimize without data
   - Slow query? Get `EXPLAIN ANALYZE` output
   - High memory? Get heap profile or container stats
   - High CPU? Get top process or flame graph if available
2. Identify bottleneck type:
   - **DB** — missing index, N+1, full table scan, lock contention
   - **App** — tight loop, blocking I/O, memory leak, inefficient algorithm
   - **Infra** — underpowered instance, network latency, cold starts
3. Root cause + fix with expected improvement
4. Wait for approval, then implement

---

## logs \<file\> (fork)
Use when user shares a log file path for analysis.

1. Check file size — if >1000 lines, read tail first (`tail -n 500`), then scan for errors
2. Categorize findings:
   - **Errors** — exceptions, failures, crashes (with timestamps)
   - **Warnings** — deprecations, retries, near-limit conditions
   - **Anomalies** — unexpected patterns, missing expected events, gaps in flow
   - **Performance** — slow queries, timeouts, high latency entries
3. Report: critical issues → recurring patterns → timeline → suggested actions
4. Include frequency for recurring items. Signal over noise.
