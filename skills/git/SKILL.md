---
name: git
description: Git workflows — commit, branch, PR, conflict resolution, init, remote. Use when performing any git operation.
allowed-tools: Bash(git *), Bash(gh *)
---

Handle git operation based on $ARGUMENTS:

**First**: detect default branch — run `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'`. If that fails, check which of `main` or `master` exists locally. Use the result as DEFAULT_BRANCH throughout.

## commit
1. Run `git diff --cached --stat` to see staged files
2. If nothing staged, run `git add -p` interactively
3. Write commit message following conventional commits: `type(scope): description`
   - Types: feat, fix, refactor, style, perf, test, chore, docs, ci
   - Scope: optional, the module/area affected
   - Description: imperative mood, lowercase, no period, under 72 chars
4. Run `git commit -m "message"`

## branch <n>
1. Run `git checkout DEFAULT_BRANCH && git pull origin DEFAULT_BRANCH`
2. Create branch: `git checkout -b <type>/<n>`
   - Types: feat/, fix/, refactor/, chore/, docs/
3. Confirm branch created with `git branch --show-current`

## pr
1. Run `git log DEFAULT_BRANCH..HEAD --oneline` to see commits
2. Run `git diff DEFAULT_BRANCH --stat` to see changed files
3. Generate PR description with:
   - Title (from branch name or commits)
   - ## What — brief summary of changes
   - ## Why — motivation/context
   - ## How — implementation approach
   - ## Testing — what was tested
4. If `gh` CLI available, offer to create PR with `gh pr create`

## conflict
1. Run `git status` to find conflicted files
2. Read each conflicted file
3. Explain both sides of each conflict clearly
4. Suggest resolution with reasoning
5. After user approves, apply fix and run `git add <file>`

## init
1. Run `git init`
2. Create appropriate `.gitignore` based on project files detected
3. Create initial commit: `chore: initial commit`
4. Ask if user wants to add a remote — if yes, follow `remote` flow

## remote <url>
1. Run `git remote -v` to check existing remotes
2. If no remote exists: `git remote add origin <url>`
3. If origin already exists and user wants to change: `git remote set-url origin <url>`
4. Verify with `git remote -v`
5. If fresh repo, offer to push: `git push -u origin DEFAULT_BRANCH`