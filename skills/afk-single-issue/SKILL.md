---
name: afk-single-issue
description: "Autonomously take conversation context or a single issue end-to-end without human input: synthesize to-spec, create/label ready-for-agent, implement in an isolated worktree via TDD, create PR with /pr, watch CI until green, squash-merge, and clean up worktree."
argument: "[optional: issue number/URL or task summary; default: synthesize from active conversation]"
---

# AFK Single Issue

Autonomous end-to-end execution of a single task or issue. Takes conversational requirements (e.g. immediately following a discussion or `/grill-with-docs`) or an existing issue/task description, formalizes the spec, creates a worktree, implements using TDD, opens a PR, watches CI until green, squash-merges, syncs `main`, and deletes the worktree. Fully hands-free and AFK.

## Operating Principles (Hard Rules)

1. **Fully AFK execution.** Do not prompt the user for seam confirmation, PR approval, or manual merge decisions during the run. The design and requirements have already been established prior to calling this skill.
2. **Strict worktree isolation.** All implementation work must occur in an isolated git worktree (`.worktrees/issue-<N>`) on branch `agent/issue-<N>` branched from the latest `origin/main`. Never modify code directly on `main` in the primary workspace.
3. **Spec-first discipline.** Every implementation must be grounded in a concrete specification using the `to-spec` format (Problem Statement, Solution, User Stories, Implementation Decisions, Testing Decisions).
4. **TDD at pre-agreed seams.** Work must follow test-driven development (red-green-refactor) as specified in `AGENTS.md` and `skill://tdd`. Run typechecks and tests regularly.
5. **Standards-compliant PR.** The PR body must follow the structure defined in `skill://pr` (Summary with diff/diagram sketch, Evidence with before/after test output, Merge Danger door & blast radius) and include `Closes #<N>`.
6. **Watch CI & auto-fix.** Monitor checks with `gh pr checks <pr> --watch`. Fix any CI failure on the same branch within the worktree (up to 3 fix cycles). If still blocked after 3 cycles, stop, leave the worktree intact, and report `blocked`.
7. **Squash-merge & main sync.** Once all required checks are green, squash-merge the PR (`gh pr merge <pr> --squash --admin`), verify the issue auto-closes, and sync `main` in the primary checkout (`git pull --ff-only`).
8. **Scoped worktree teardown.** Upon successful merge or early resolution, remove all worktrees matching issue `<N>` (`git worktree list | grep -E "issue[-_]$N\b"`) and delete associated local branches (`*issue*$N*`). If blocked or failed, preserve them for developer inspection. Never touch unrelated worktrees belonging to other issues.

---

## Phase 0 — Input Synthesis & Spec Formulation

### Step 0.1: Resolve Target Issue or Context

Determine the input mode based on provided arguments and current conversation:

- **Mode A: Issue argument provided (`#<N>` or issue URL)**
  - Fetch issue details: `gh issue view <N> --json number,title,body,labels,state`.
  - Check if issue is already resolved/merged: `gh pr list --state all --search "$N"`.
    - If a matching PR is already merged:
      - Ensure issue is closed: if state is `OPEN`, run `gh issue close <N> --comment "Resolved and merged to main."`.
      - Sync `main` in the primary workspace (`git pull --ff-only origin main`).
      - Teardown any lingering worktrees or local branches matching issue $N$ (see Phase 4 Step 5).
      - Report outcome and exit early.
  - If the issue body already contains a spec (`to-spec` or `to-tickets` markers), proceed directly to Phase 1.
  - If the issue body is raw prose or unlabeled:
    - Generate a formal spec following the `<spec-template>` below.
    - Update the issue body: `gh issue edit <N> --body "<spec>"` and add label `ready-for-agent`.
  - Record issue number $N$.

- **Mode B: No issue argument provided (Conversation Synthesis / Post-Grill)**
  - Synthesize the active conversation, decisions made during `/grill-with-docs`, and codebase context into a formal spec.
  - Identify the seams at which the feature will be tested (highest seam possible, preferring existing seams over new ones).
  - Draft the issue title in conventional-commit format (e.g. `feat: add user authentication flow`).
  - Create the GitHub issue:
    ```bash
    gh issue create --title "<title>" --body "<spec>" --label "ready-for-agent"
    ```
  - Parse the newly created issue number $N$.

### Spec Template (`to-spec`)

```markdown
## Problem Statement
<The problem that the user is facing, from the user's perspective>

## Solution
<The solution to the problem, from the user's perspective>

## User Stories
1. As an <actor>, I want a <feature>, so that <benefit>
<Cover all relevant user flows and edge cases>

## Implementation Decisions
- Modules to be built / modified
- Interfaces & contracts modified
- Architectural decisions & schema changes
- (Do NOT include fragile file paths unless referencing an existing prototype snippet)

## Testing Decisions
- Seam selection & description of good test behavior
- Modules to be tested
- Prior art / test conventions in the repo

## Out of Scope
<Explicitly out-of-scope items>
```

---

## Phase 1 — Grounding & Worktree Setup

1. **Repo profile check**:
   - Check repo documentation: `AGENTS.md`, `CONTEXT.md`, `docs/adr/`, `docs/agents/*.md`.
   - Determine required checks: inspect `.github/workflows/*` and branch protection/rulesets (`gh api repos/<OWNER>/<REPO>/rules/branches/main`).
   - Merge policy: squash-merge (`--squash --admin`).
2. **Fetch and branch**:
   ```bash
   git fetch origin main
   ```
3. **Create isolated worktree**:
   - Path: `.worktrees/issue-<N>`
   - Branch: `agent/issue-<N>`
   ```bash
   git worktree add -b "agent/issue-$N" ".worktrees/issue-$N" origin/main
   ```
4. **Environment preparation**:
   - Inside `.worktrees/issue-<N>`, check if dependencies are installed (e.g. `npm ci`, `flutter pub get`, etc.) if required by the repo.

---

## Phase 2 — Implementation & Code Review

Execute inside `.worktrees/issue-<N>`:

1. **TDD Cycle (`skill://tdd`)**:
   - Write failing tests first at the defined seams.
   - Implement minimal code to pass tests.
   - Refactor cleanly preserving readability and domain conventions.
2. **Quality Gates**:
   - Run typechecking, linting, and full test suite as defined in the repo (`AGENTS.md` / `package.json`):
     ```bash
     npm run typecheck && npm run lint && npm test
     ```
   - All tests must pass locally before creating a PR.
3. **Self-Review (`skill://code-review`)**:
   - Run code-review against standard axes (Standards & Spec alignment).
   - Ensure changes strictly adhere to the spec decisions and no extraneous modifications or regressions are introduced.
4. **Commit**:
   - Commit changes with conventional-commit message referencing the issue:
     ```bash
     git add -A
     git commit -m "feat: <description> (closes #$N)"
     ```

---

## Phase 3 — Pull Request & CI Watch

1. **Push branch**:
   ```bash
   git push origin "agent/issue-$N"
   ```
2. **Draft PR (`skill://pr`)**:
   - Create PR using GitHub CLI. Body must use the standard template:
     ```bash
     gh pr create --base main --head "agent/issue-$N" --title "<conventional title>" --body "$(cat <<'EOF'
     ## Summary
     <ascii diagram, call tree, or diff sketch>

     ## Evidence
     - **Before:** <failing test output or baseline behavior>
     - **After:** <passing test output and verified functionality>

     ## Merge Danger
     **Door:** <one-way or two-way>
     **Blast Radius:** <narrow | isolated | broad>

     Closes #<N>
     EOF
     )"
     ```
   - Capture PR number `<PR>`.
3. **Watch CI checks**:
   ```bash
   gh pr checks "$PR" --watch
   ```
4. **Auto-fix cycle (up to 3 attempts)**:
   - If any check fails:
     - Inspect failure logs: `gh pr checks "$PR"` or `gh run view <run-id> --log-failed`.
     - Apply fixes inside `.worktrees/issue-<N>`.
     - Re-run local gate commands.
     - Commit and push to `agent/issue-<N>`.
     - Re-watch CI.
   - If checks fail >3 attempts or external blockers arise, STOP, label issue `blocked`, leave worktree intact, and report failure details.

---

## Phase 4 — Squash-Merge & Teardown

1. **Verify checks**: Confirm all required CI checks are green on GitHub.
2. **Squash-merge**:
   ```bash
   gh pr merge "$PR" --squash --admin
   ```
3. **Verify closure**:
   - Check that PR status is `MERGED`: `gh pr view "$PR" --json state`.
   - Confirm issue #$N$ is closed; if still open, close it: `gh issue close "$N" --comment "Resolved and merged via PR #$PR."`.
4. **Sync main**:
   - In the primary workspace:
     ```bash
     git pull --ff-only origin main
     ```
5. **Scoped worktree teardown**:
   - Remove all worktrees associated with issue $N$:
     ```bash
     git worktree list | grep -E "issue[-_]$N\b" | awk '{print $1}' | while read -r wt; do
       git worktree remove --force "$wt"
     done
     ```
   - Delete all matching local feature branches:
     ```bash
     git branch --list "*issue*$N*" --format="%(refname:short)" | while read -r br; do
       git branch -D "$br"
     done
     ```
   - Prune worktree metadata:
     ```bash
     git worktree prune
     ```
6. **Report outcome**:
   Output final summary:
   ```
   ISSUE <N>: merged (PR #<PR>)
   Title: <PR title>
   Branch: agent/issue-<N> (cleaned up)
   Worktree: .worktrees/issue-<N> (removed)
   Main: synchronized with origin/main
   ```
