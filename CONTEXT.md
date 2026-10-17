# AFK Skills Suite

Autonomous GitHub development workflows and skills for AI coding agents.

## Suite Skills

1. **`afk-issue-loop`**: Autonomous batch processor over open GitHub issues (triage, implement, PR, watch CI, squash-merge in serial or parallel mode).
2. **`afk-single-issue`**: Autonomous end-to-end runner for a single task (post-grill or specific issue): synthesizes `to-spec`, creates/labels issue, implements via TDD in an isolated worktree, opens PR with `/pr`, monitors CI until green, squash-merges, and tears down the worktree.
3. **`afk-ship`**: Autonomous runner for completed work: takes an existing worktree/branch where implementation is finished, verifies gates/commits, pushes, opens PR with `/pr`, monitors CI until green (auto-fix up to 3 cycles), squash-merges, syncs `main`, and deletes the worktree.

## Language

**Worktree Registry**:
The in-memory set of git worktree paths explicitly created and tracked by the current orchestrator session (`.worktrees/issue-<N>`).
_Avoid_: Global worktrees, existing worktree scan

**Settled Worktree**:
An isolated git worktree created for an issue that has completed its lifecycle (merged and synced to main, or closed) and is safe for teardown.
_Avoid_: Abandoned worktree, stale worktree

**Foreign Worktree**:
Any git worktree not created by the active orchestrator run, including worktrees created by other agents, previous sessions, or human developers.
_Avoid_: Orphan worktree, external branch
