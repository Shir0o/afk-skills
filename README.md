# afk-skills

> Autonomous, cost-minimal GitHub workflows for AI coding agents. Run batch issue loops or ship single tasks hands-free from specification to merge.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## Skills in this Suite

This repository provides complementary autonomous skills:

| Skill | Path | Description |
|---|---|---|
| **`afk-single-issue`** | [`skills/afk-single-issue/SKILL.md`](skills/afk-single-issue/SKILL.md) | **Single-task end-to-end runner**: Takes an active conversation (e.g. post-`/grill-with-docs`) or single issue, synthesizes `to-spec`, labels `ready-for-agent`, implements via TDD in an isolated worktree, opens PR with `/pr`, watches CI, squash-merges, and cleans up worktree. |
| **`afk-ship`** | [`skills/afk-ship/SKILL.md`](skills/afk-ship/SKILL.md) | **Fast-track PR & merge runner**: For when implementation is already finished in a worktree or branch. Verifies commits/gates, pushes branch, drafts PR via `/pr`, watches CI (auto-fixing up to 3 cycles), squash-merges, syncs `main`, and deletes worktree. |
| **`afk-issue-loop`** | [`skills/afk-issue-loop/SKILL.md`](skills/afk-issue-loop/SKILL.md) | **Backlog batch loop**: Continuously processes issues across your GitHub repository (triages unlabeled issues, implements `ready-for-agent` issues in serial or parallel worktrees, PRs, watches CI, squash-merges, and repeats). |
| **`afk-ui-ux-loop`** | [`skills/afk-ui-ux-loop/SKILL.md`](skills/afk-ui-ux-loop/SKILL.md) | **Visual UI/UX & E2E audit loop**: Audits a single repo or rotates across multi-repo workspaces via LRU state, runs multi-viewport screenshot & interactive click-through checks (Flutter widget goldens / optional emulator or Web DevTools MCP), ships one surgical UI/UX fix via PR, watches CI, squash-merges, and advances Release Please workflows when present. |
| **`afk-email-triage`** | [`skills/afk-email-triage/SKILL.md`](skills/afk-email-triage/SKILL.md) | **Tech alert email triage & remediation**: Scans Gmail inbox for unaddressed tech alerts (GitGuardian, Sentry, CI/security), maps each to its local repo, implements fixes in isolated worktrees (optionally via `--code-cmd`), opens PRs, watches CI, squash-merges, and archives resolved emails. |

---

## 1. `afk-single-issue` (Post-Grill & Single Issue Runner)

Use this immediately after a planning session, interactive `/grill-with-docs`, or when targeting a specific feature task.

### Workflow

```
Active Chat / /grill-with-docs (Clarification complete)
  │
  ▼
Phase 0: Synthesize to-spec
  • Formulate Problem Statement, Solution, User Stories, Decisions, Seams
  • Create or update GitHub issue with label `ready-for-agent`
  │
  ▼
Phase 1: Isolated Worktree
  • Create .worktrees/issue-<N> on agent/issue-<N> from origin/main
  │
  ▼
Phase 2: Implement (TDD + Review)
  • TDD at pre-agreed seams (/tdd)
  • Full local test gate & /code-review
  │
  ▼
Phase 3: Pull Request & CI Watch
  • Create PR using /pr template (Summary sketch, Evidence, Merge Danger)
  • Watch CI (gh pr checks --watch), auto-fix up to 3 cycles if red
  │
  ▼
Phase 4: Squash-Merge & Teardown
  • gh pr merge --squash --admin
  • Sync main checkout (git pull --ff-only)
  • Remove .worktrees/issue-<N> and delete agent/issue-<N> branch
```

### Usage

```bash
# Run on current chat context (e.g. right after /grill-with-docs)
cook this
run afk-single-issue

# Or run on a specific issue
run afk-single-issue for #42
```

---

## 2. `afk-ship` (Implementation Done Runner)

Use this when implementation is already finished in a worktree or feature branch, and you just want to ship it hands-free through PR, CI, squash-merge, and worktree cleanup.

### Workflow

```
Completed Implementation in Worktree / Branch
  │
  ▼
Phase 0: Target Context Discovery
  • Detect active worktree & branch (from inside worktree or via issue #/branch arg)
  • Resolve linked issue # (from branch, commits, or args)
  │
  ▼
Phase 1: Pre-Push Integrity
  • Stage & commit any remaining uncommitted changes
  • Run local quality gates (typecheck, lint, test)
  • Push branch to remote (git push -u origin <branch>)
  │
  ▼
Phase 2: Pull Request & CI Watch
  • Create PR using /pr template (Summary sketch, Evidence, Merge Danger)
  • Watch CI (gh pr checks --watch), auto-fix up to 3 cycles if red
  │
  ▼
Phase 3: Squash-Merge & Clean Teardown
  • gh pr merge --squash --admin
  • Navigate to primary repo root & sync main (git pull --ff-only)
  • Remove worktree, delete local branch, and prune metadata
```

### Usage

```bash
# Run from inside the worktree when implementation is done:
run afk-ship
ship this

# Or run from the primary repo targeting an issue or branch:
run afk-ship for #42
run afk-ship agent/issue-42
```

---

## 3. `afk-issue-loop` (Autonomous Backlog Processor)

Use this to autonomously drain an entire repository queue.

- **Triage** — labels unlabeled issues (`needs-triage → needs-info / ready-for-agent / wontfix`), closes exact duplicates inline.
- **Implement** — branches off `main`, writes code TDD-style, opens a PR, watches CI, fixes failures, and merges.
- **Orchestrate** — runs serially (1 subagent at a time) or in parallel ($N$ concurrent workers partitioned across disjoint subsystems).
- **Interruption Resilience** — resumes in-flight subagents by inspecting session history and worktree diffs.
- **Scoped Cleanup** — cleans up only worktrees registered to the active session.

### Usage

```bash
run the issue loop
process open issues AFK
run the issue loop in parallel
```

---

## 4. `afk-ui-ux-loop` (Visual UI/UX & E2E Audit Loop)

Use this to inspect rendered screens across viewports and interactive states, fix one high-impact usability or visual defect, and ship it through CI and merge.

- **Single-Repo or Fleet Rotation** — defaults to the current repository; pass `--root <dir>` to discover Flutter/Web repos and rotate through the oldest `--limit <N>` repos using `--state-file`.
- **Visual + Click-Through Verification** — captures Light, Dark, and Small + `1.6x` text-scale views (`390x844`, `320x568`) plus navigation tap/click flows for Flutter (`flutter test --update-goldens`) and Mobile/Desktop (`375x667`, `1280x800`) via `chrome-devtools-mcp` for Web.
- **Optional External Code Delegation** — works standalone or delegates heavy code/test edits via `--code-cmd` while keeping screenshot inspection in the multimodal subagent.
- **Clean Teardown & Release** — deletes temporary screenshots, shuts down any temporary emulator/dev server, opens a PR, watches CI, squash-merges, and optionally advances `release-please.yml` if configured.

### Usage

```bash
# Audit the current repository
run afk-ui-ux-loop

# Rotate through the 5 oldest UI repos under ~/projects
run afk-ui-ux-loop --root ~/projects --limit 5

# Audit a single repo and delegate code edits to an external CLI
run afk-ui-ux-loop --repo my-app --code-cmd "opencode run -m <model>"
```

---

## 5. `afk-email-triage` (Tech Alert Email Triage & Remediation)

Use this to scan your Gmail inbox for unaddressed technical alerts (GitGuardian, Sentry, GitHub CI/security, cloud alerts), fix them in isolated worktrees, ship PRs through CI to squash-merge, and archive resolved emails.

- **Deduplication & Already-Addressed Check** — groups duplicate alert emails per `(repo, issue)` and verifies on `main`/open PRs before writing code.
- **Secret-Safe Handling** — never copies raw secrets from GitGuardian alerts into coding CLI prompts, commits, or PRs, and flags exposed credentials for manual rotation.
- **Optional External Code Delegation** — supports `--code-cmd "opencode run -m <model>"` or native editing tools, with lightweight local gates and GitHub CI verification.
- **Archive on Resolution** — archives emails via `gmail mutate archive` only after the PR merges or the issue is verified already resolved.

### Usage

```bash
# Triage inbox tech alerts across repos under ~
run afk-email-triage --root ~ --code-cmd "opencode run -m opencode-go/deepseek-v4.1-flash" --admin-merge

# Preview matches without editing or archiving
run afk-email-triage --root ~ --dry-run
```

---

## Dependencies

These workflows integrate with companion skill files from [Matt Pocock's engineering skill suite](https://github.com/mattpocock/skills):

| Skill | Used by | Purpose |
|---|---|---|
| [`to-spec`](https://github.com/mattpocock/skills/tree/main/to-spec) | `afk-single-issue` | Synthesizes conversation context into structured spec |
| [`triage`](https://github.com/mattpocock/skills/tree/main/triage) | `afk-issue-loop` | Labels unlabeled/needs-triage issues, closes duplicates |
| [`implement`](https://github.com/mattpocock/skills/tree/main/implement) | `afk-single-issue`, `afk-issue-loop` | Branches, writes code, prepares PR |
| [`tdd`](https://github.com/mattpocock/skills/tree/main/tdd) | `afk-single-issue`, `afk-issue-loop` | Red-green-refactor cycle |
| [`code-review`](https://github.com/mattpocock/skills/tree/main/code-review) | `afk-single-issue`, `afk-issue-loop` | Pre-push self-review against standards & spec |
| [`pr`](https://github.com/mattpocock/skills/tree/main/skills/engineering/pr) | All | Standards-compliant PR description |

---

## Installation

### 1. Install companion skills

```bash
for skill in to-spec triage implement tdd code-review pr; do
  mkdir -p .agents/skills/$skill
  curl -fsSL "https://raw.githubusercontent.com/mattpocock/skills/main/skills/engineering/$skill/SKILL.md" \
    -o .agents/skills/$skill/SKILL.md
done
```

### 2. Install AFK Skills

```bash
# Install afk-ship
mkdir -p .agents/skills/afk-ship
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-skills/main/skills/afk-ship/SKILL.md \
  -o .agents/skills/afk-ship/SKILL.md

# Install afk-single-issue
mkdir -p .agents/skills/afk-single-issue
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-skills/main/skills/afk-single-issue/SKILL.md \
  -o .agents/skills/afk-single-issue/SKILL.md

# Install afk-issue-loop
mkdir -p .agents/skills/afk-issue-loop
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-skills/main/skills/afk-issue-loop/SKILL.md \
  -o .agents/skills/afk-issue-loop/SKILL.md

# Install afk-ui-ux-loop
mkdir -p .agents/skills/afk-ui-ux-loop
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-skills/main/skills/afk-ui-ux-loop/SKILL.md \
  -o .agents/skills/afk-ui-ux-loop/SKILL.md

# Install afk-email-triage
mkdir -p .agents/skills/afk-email-triage
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-skills/main/skills/afk-email-triage/SKILL.md \
  -o .agents/skills/afk-email-triage/SKILL.md
```

---

## Optional repo docs respected across skills

| File | Purpose |
|---|---|
| `docs/agents/issue-tracker.md` | Custom label vocabulary, PR-as-request-surface flag |
| `docs/agents/triage-labels.md` | Label definitions |
| `AGENTS.md` | Coding standards, full-gate commands (`typecheck && lint && test && build`) |
| `CONTEXT.md` | Domain model / codebase orientation |
| `docs/adr/` | Architecture decisions |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).
