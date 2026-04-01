---
name: git
description: Git workflows — commit, branch, PR, conflict resolution, init, remote, stash, undo, sync, tag, push, review. Use when performing any git operation.
---

Handle git operation based on $ARGUMENTS:

**First**: detect default branch — run `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'`. If that fails, check which of `main` or `master` exists locally. Use the result as DEFAULT_BRANCH throughout.

---

## commit
1. Run `git diff --cached --stat` to see staged files
2. If nothing staged:
   - Run `git status` to list unstaged/untracked files
   - Run `git diff --stat` to see what changed
   - Stage relevant files with `git add <file> ...` (never use `git add -A` blindly — skip secrets, binaries, unrelated files)
3. Run `git diff --cached` to read the actual changes
4. Write a commit message from the diff following conventional commits: `type(scope): description`
   - Types: feat, fix, refactor, style, perf, test, chore, docs, ci
   - Scope: optional, the module/area affected
   - Description: imperative mood, lowercase, no period, under 72 chars
5. Commit using HEREDOC to handle special characters:
   ```
   git commit -m "$(cat <<'EOF'
   type(scope): description
   EOF
   )"
   ```

---

## branch <name>
1. Run `git checkout DEFAULT_BRANCH && git pull origin DEFAULT_BRANCH`
2. Create branch: `git checkout -b <type>/<name>`
   - Types: feat/, fix/, refactor/, chore/, docs/
   - `<name>` is a short kebab-case slug, e.g. `add-auth`, `fix-login-bug`
   - If an issue number is provided, prefix it: `feat/123-add-oauth`
3. Confirm with `git branch --show-current`

---

## pr
1. Run `git log DEFAULT_BRANCH..HEAD --oneline` to see commits
2. Run `git diff DEFAULT_BRANCH --stat` to see changed files
3. Check if branch is pushed: `git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null`
   - If no upstream set, run `git push -u origin HEAD` first
4. Generate PR description with:
   - Title (from branch name or commits, under 70 chars)
   - ## What — brief summary of changes
   - ## Why — motivation/context
   - ## How — implementation approach
   - ## Testing — what was tested
5. If `gh` CLI available, create PR:
   ```
   gh pr create --title "..." --assignee @me --base DEFAULT_BRANCH --body "$(cat <<'EOF'
   ## What
   ...
   ## Why
   ...
   ## How
   ...
   ## Testing
   ...
   EOF
   )"
   ```

---

## push
1. Get current branch: `git branch --show-current`
2. Warn and confirm if branch is `main` or `master` — pushing directly to default branch is risky
3. Check if upstream is set: `git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null`
   - If no upstream: `git push -u origin <current-branch>`
   - If upstream exists: `git push`
4. Confirm with output from push

---

## conflict
1. Run `git status` to find conflicted files and detect context (merge vs rebase)
2. Read each conflicted file
3. Explain both sides of each conflict clearly (ours vs theirs)
4. Suggest resolution with reasoning
5. After user approves, apply fix and run `git add <file>`
6. Once all conflicts are resolved:
   - If rebase in progress: `git rebase --continue`
   - If merge in progress: `git merge --continue`
   - Check with `git status` to confirm clean state

---

## stash
Sub-commands based on argument:

- **`save <msg>`** — `git stash push -m "<msg>"` to save with a descriptive label
- **`pop`** — `git stash pop` to apply and remove the latest stash
- **`apply <n>`** — `git stash apply stash@{<n>}` to apply without removing
- **`drop <n>`** — `git stash drop stash@{<n>}` to discard a specific stash
- **`list`** (default if no sub-command) — `git stash list` to show all stashes

Always run `git stash list` before pop/apply/drop so the user can see what's available.

---

## undo
1. Run `git log --oneline -5` to show recent commits
2. Ask or infer which commit to undo (default: HEAD)
3. Explain options and confirm with user:
   - `--soft`: undo commit, keep changes **staged** (safe, just moves HEAD)
   - `--mixed` (default): undo commit, keep changes **unstaged**
   - `--hard`: undo commit and **discard all changes** — irreversible, confirm explicitly
4. Run `git reset <mode> HEAD~1` (or appropriate ref)
5. Run `git status` to confirm the result

---

## sync
Bring the current branch up to date with DEFAULT_BRANCH.

1. Run `git fetch origin`
2. Run `git rebase origin/DEFAULT_BRANCH`
3. If conflicts arise during rebase, follow the `conflict` flow to resolve them
4. Confirm with `git log --oneline -5` to show current state

---

## tag
Sub-commands based on argument:

- **`list`** (default) — `git tag -l` to list all tags
- **`create <version> <msg>`** — `git tag -a <version> -m "<msg>"` (use semver, e.g. `v1.2.0`)
- **`push <version>`** — `git push origin <version>` to push a specific tag
- **`push-all`** — `git push origin --tags` to push all tags
- **`delete <version>`** — `git tag -d <version>` (and optionally `git push origin :refs/tags/<version>` to remove remote)

---

## init
1. Run `git init`
2. Detect project type from files present:
   - `requirements.txt` / `pyproject.toml` / `*.py` → Python .gitignore
   - `package.json` → Node .gitignore
   - `go.mod` → Go .gitignore
   - Multiple types → combine relevant sections
3. Write appropriate `.gitignore`
4. Create initial commit: `chore: initial commit`
5. Ask if user wants to add a remote — if yes, follow `remote` flow

---

## review
Review only the current diff — staged changes, a PR branch, or a specific commit. Scope is the delta, not the whole codebase.

1. Determine what to review:
   - No argument: `git diff DEFAULT_BRANCH...HEAD` (current branch vs base)
   - Staged only: `git diff --cached`
   - Specific commit: `git show <ref>`
2. Read the changed files at the affected lines for full context
3. Report findings grouped by category:
   - **Correctness** — logic errors, off-by-one, wrong conditions, missing cases
   - **Security** — any input reaching DB/shell/template unvalidated, secrets, missing auth
   - **Breaking changes** — API changes, removed exports, schema changes without migration
   - **Code quality** — DRY violations, unclear naming, missing type annotations, dead code
   - **Tests** — changed logic with no corresponding test update
4. For each finding: file + line, what's wrong, recommended fix (one sentence)
5. End with: overall verdict — **Approve** / **Approve with minor comments** / **Request changes**

Rules: review the diff, not existing code. Don't flag pre-existing issues outside the changed lines. Be specific — no generic advice.

---

## remote <url>
1. Run `git remote -v` to check existing remotes
2. If no remote exists: `git remote add origin <url>`
3. If origin already exists and user wants to change: `git remote set-url origin <url>`
4. Verify with `git remote -v`
5. If fresh repo, offer to push: `git push -u origin DEFAULT_BRANCH`
