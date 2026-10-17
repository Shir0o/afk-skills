---
name: afk-ship
description: "Autonomously ship completed implementation from an isolated worktree or branch: verify gates/commits, push branch, draft standards-compliant PR via skill://pr, watch CI until green (auto-fix up to 3 attempts), squash-merge to main, sync main checkout, and cleanly teardown the worktree."
argument: "[optional: issue number/URL, branch name, or worktree path; default: detect active worktree or current branch]"
---

# AFK Ship

Autonomous runner for shipping completed implementations. When coding, refactoring, or bugfixing is already finished in a git worktree or feature branch, `afk-ship` takes over the entire release tail end: verifies uncommitted changes and gates, pushes the branch, drafts a standards-compliant pull request using `skill://pr`, monitors CI until green (with up to 3 auto-fix cycles), squash-merges into `main`, synchronizes the primary checkout, and safely tears down the worktree and branch. Fully hands-free and AFK.

## Operating Principles (Hard Rules)

1. **Fully AFK execution.** Do not prompt the user for seam confirmation, PR approval, or manual merge decisions during the run. Implementation is already complete.
2. **Directory & worktree awareness.** The command may be executed from inside the worktree itself OR from the primary repository checkout. Always detect the primary repository root and target worktree. **Never** attempt `git worktree remove` while current working directory is inside that worktree; navigate back to the primary checkout before teardown.
3. **Commit & pre-push integrity.** If uncommitted changes exist, stage and commit them with a descriptive conventional commit referencing the issue (e.g. `feat: ... (closes #<N>)`). Run local quality gates (typecheck, lint, tests) before pushing to remote.
4. **Standards-compliant PR via `skill://pr`.** The PR body must adhere to `skill://pr`:
   - `## Summary`: visual summary (ascii diagram, component tree, or diff sketch).
   - `## Evidence`: concrete verification (before vs. after test runs or output).
   - `## Merge Danger`: Door (`one-way` or `two-way`) and Blast Radius (`narrow`, `isolated`, or `broad`).
   - `Closes #<N>` if associated with an issue.
5. **Watch CI & auto-fix.** Monitor checks with `gh pr checks "$PR" --watch`. If any check fails, inspect logs (`gh pr checks "$PR"` or `gh run view <run-id> --log-failed`), apply fixes in the worktree, re-run local gates, commit, push, and re-watch (up to 3 fix cycles). If still red after 3 cycles, halt, preserve the worktree for inspection, and report `blocked`.
6. **Squash-merge & main sync.** Once all required checks are green, squash-merge the PR (`gh pr merge "$PR" --squash --admin`), verify the PR is merged and associated issue closed, and sync `main` in the primary checkout (`git pull --ff-only origin main`).
7. **Scoped worktree teardown.** Upon successful merge, switch to the primary repository root, remove the specific worktree (`git worktree remove --force <path>`), delete the local branch (`git branch -D <branch>`), and prune metadata (`git worktree prune`). Never delete unrelated worktrees or worktrees of failed/blocked runs.

---

## Phase 0 — Target Context Discovery

Determine the target worktree, branch, primary checkout root, and linked issue:

1. **Locate primary repository root and worktree context**:
   ```bash
   # Check if inside a worktree
   PRIMARY_ROOT=$(git rev-parse --path-format=absolute --git-common-dir | sed 's/\/\.git$//')
   CURRENT_ROOT=$(git rev-parse --show-toplevel)
   ```
   - If `CURRENT_ROOT` differs from `PRIMARY_ROOT`, execution was triggered from **inside a worktree**. Target that worktree and its active branch.
   - If `CURRENT_ROOT` equals `PRIMARY_ROOT`, resolve target from argument:
     - **Mode A: Issue argument (`#<N>` or `<N>`)**:
       Target worktree `.worktrees/issue-<N>` or branch `agent/issue-<N>`.
     - **Mode B: Worktree path or branch argument**:
       Target the specified path or branch.
     - **Mode C: No argument**:
       Inspect `git worktree list`. Check `.worktrees/`, `~/.gemini/antigravity/worktrees/`, or active project worktrees. If active worktrees exist, target the one matching the current task/issue or most recently modified; if none, target the current feature branch.

2. **Extract branch and linked issue number**:
   - Determine target branch: `git -C "$WORKTREE_DIR" rev-parse --abbrev-ref HEAD`.
   - Determine linked issue number $N$ from:
     - Argument if provided.
     - Branch name (e.g. `agent/issue-42` -> `42`, `issue-105` -> `105`).
     - Recent commit messages (`git -C "$WORKTREE_DIR" log -n 5 --oneline | grep -oE "#[0-9]+" | head -n1`).

3. **Check already-merged status**:
   - Check if a matching PR is already merged:
     ```bash
     gh pr list --state all --head "$BRANCH" --json number,state,mergedAt
     ```
   - If already merged:
     - If linked issue is still open, close it: `gh issue close "$N" --comment "Resolved and merged to main."`.
     - Switch to primary checkout: `cd "$PRIMARY_ROOT" && git pull --ff-only origin main`.
     - Remove worktree: `git worktree remove --force "$WORKTREE_DIR" 2>/dev/null || true`.
     - Delete branch: `git branch -D "$BRANCH" 2>/dev/null || true`.
     - Report outcome and exit early.

---

## Phase 1 — Pre-Push Verification & Commit

Execute inside the target worktree directory (`$WORKTREE_DIR`):

1. **Check working tree status**:
   ```bash
   git status --short
   ```
   - If uncommitted/unstaged changes exist:
     - Stage changes: `git add -A`.
     - Commit with conventional message:
       ```bash
       if [ -n "$N" ]; then
         git commit -m "feat: complete implementation (closes #$N)"
       else
         git commit -m "feat: complete implementation"
       fi
       ```

2. **Run local quality gates**:
   - Inspect repo guidelines (`AGENTS.md`, `package.json`, `Makefile`, etc.).
   - Run typechecking, linting, and tests (e.g. `npm run typecheck && npm run lint && npm test` or `flutter analyze && flutter test`).
   - If local tests fail, fix them inside the worktree before pushing.

3. **Push branch to remote**:
   ```bash
   git push -u origin "$BRANCH"
   ```

---

## Phase 2 — Draft Pull Request & CI Watch

1. **Check for existing PR**:
   ```bash
   PR_JSON=$(gh pr view "$BRANCH" --json number,url,state 2>/dev/null || true)
   ```
   - If an open PR already exists, capture `PR` number and URL.

2. **Draft PR via `skill://pr` (if not already opened)**:
   - Generate summary diff/diagram sketch, evidence before/after, and merge danger assessment.
   - Create PR:
     ```bash
     gh pr create --base main --head "$BRANCH" --title "<conventional commit title>" --body "$(cat <<'EOF'
     ## Summary
     <ascii diagram, call tree, component tree, or diff sketch>

     ## Evidence
     - **Before:** <baseline or prior behavior>
     - **After:** <passing test output and verified functionality>

     ## Merge Danger
     **Door:** <one-way or two-way>
     **Blast Radius:** <narrow | isolated | broad>

     Closes #<N>
     EOF
     )"
     ```
   - Capture `PR` number.

3. **Watch CI checks**:
   ```bash
   gh pr checks "$PR" --watch
   ```

4. **Auto-fix cycle (up to 3 attempts)**:
   - If any check fails:
     - Inspect failure logs: `gh pr checks "$PR"` or `gh run view --log-failed`.
     - Apply fixes inside `$WORKTREE_DIR`.
     - Re-run local gate commands.
     - Commit and push:
       ```bash
       git add -A
       git commit -m "fix(ci): resolve test/build check failures"
       git push origin "$BRANCH"
       ```
     - Re-watch CI checks: `gh pr checks "$PR" --watch`.
   - If checks continue to fail after 3 attempts:
     - STOP.
     - If linked issue exists, label it `blocked`: `gh issue edit "$N" --add-label "blocked" 2>/dev/null || true`.
     - Leave the worktree intact for developer inspection.
     - Report failure details and exit.

---

## Phase 3 — Squash-Merge & Clean Teardown

1. **Verify all checks passed**:
   - Ensure all required status checks report success on GitHub.

2. **Squash-merge**:
   ```bash
   gh pr merge "$PR" --squash --admin
   ```

3. **Verify merge and issue closure**:
   - Verify PR state: `gh pr view "$PR" --json state --jq .state` (must be `MERGED`).
   - If linked issue $N$ is still open, close it:
     ```bash
     gh issue close "$N" --comment "Resolved and merged via PR #$PR."
     ```

4. **Navigate to primary repository root & sync main**:
   ```bash
   cd "$PRIMARY_ROOT"
   git pull --ff-only origin main
   ```

5. **Scoped worktree & branch teardown**:
   - Remove the worktree:
     ```bash
     git worktree remove --force "$WORKTREE_DIR" 2>/dev/null || true
     ```
   - Delete local branch:
     ```bash
     git branch -D "$BRANCH" 2>/dev/null || true
     ```
   - **Sweep stale task & merged worktrees**: Inspect `git worktree list`. If any auxiliary or session worktrees (e.g. in `~/.gemini/antigravity/worktrees/` or `.worktrees/`) were created for this task or are already merged into `main`, remove them with `git worktree remove --force <path>` and delete their local branches.
   - Prune worktree metadata and remote references:
     ```bash
     git worktree prune
     git remote prune origin
     ```

6. **Report outcome**:
   Output final summary:
   ```
   ========================================
   AFK SHIP COMPLETE
   ========================================
   PR: #<PR> (squash-merged)
   Issue: #<N> (closed)
   Branch: <BRANCH> (deleted)
   Worktree: <WORKTREE_DIR> (removed)
   Main: synchronized with origin/main
   ========================================
   ```
