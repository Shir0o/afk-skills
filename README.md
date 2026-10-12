# afk-skills

> Autonomous, cost-minimal GitHub workflows for AI coding agents. Run batch issue loops or ship single tasks hands-free from specification to merge.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## Skills in this Suite

This repository provides two complementary autonomous skills:

| Skill | Path | Description |
|---|---|---|
| **`afk-single-issue`** | [`skills/afk-single-issue/SKILL.md`](skills/afk-single-issue/SKILL.md) | **Single-task end-to-end runner**: Takes an active conversation (e.g. post-`/grill-with-docs`) or single issue, synthesizes `to-spec`, labels `ready-for-agent`, implements via TDD in an isolated worktree, opens PR with `/pr`, watches CI, squash-merges, and cleans up worktree. |
| **`afk-issue-loop`** | [`skills/afk-issue-loop/SKILL.md`](skills/afk-issue-loop/SKILL.md) | **Backlog batch loop**: Continuously processes issues across your GitHub repository (triages unlabeled issues, implements `ready-for-agent` issues in serial or parallel worktrees, PRs, watches CI, squash-merges, and repeats). |

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

## 2. `afk-issue-loop` (Autonomous Backlog Processor)

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

## Dependencies

These workflows integrate with companion skill files from [Matt Pocock's engineering skill suite](https://github.com/mattpocock/skills):

| Skill | Used by | Purpose |
|---|---|---|
| [`to-spec`](https://github.com/mattpocock/skills/tree/main/to-spec) | `afk-single-issue` | Synthesizes conversation context into structured spec |
| [`triage`](https://github.com/mattpocock/skills/tree/main/triage) | `afk-issue-loop` | Labels unlabeled/needs-triage issues, closes duplicates |
| [`implement`](https://github.com/mattpocock/skills/tree/main/implement) | Both | Branches, writes code, prepares PR |
| [`tdd`](https://github.com/mattpocock/skills/tree/main/tdd) | Both | Red-green-refactor cycle |
| [`code-review`](https://github.com/mattpocock/skills/tree/main/code-review) | Both | Pre-push self-review against standards & spec |
| [`pr`](https://github.com/mattpocock/skills/tree/main/skills/engineering/pr) | Both | Standards-compliant PR description |

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
# Install afk-single-issue
mkdir -p .agents/skills/afk-single-issue
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-skills/main/skills/afk-single-issue/SKILL.md \
  -o .agents/skills/afk-single-issue/SKILL.md

# Install afk-issue-loop
mkdir -p .agents/skills/afk-issue-loop
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-skills/main/skills/afk-issue-loop/SKILL.md \
  -o .agents/skills/afk-issue-loop/SKILL.md
```

---

## Optional repo docs respected by both skills

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
