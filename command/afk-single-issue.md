---
description: Run the AFK single issue workflow — autonomously formulate a spec from conversation context or an issue, implement in a worktree, PR, watch CI, squash-merge, and clean up.
---

Autonomously run the `afk-single-issue` skill for the current repository and task:

1. Load `skill://afk-single-issue` and execute all phases hands-free without human input.
2. If an issue number or issue URL was provided in arguments ($ARGUMENTS), fetch that issue. If already specified, proceed directly to implementation; otherwise formulate the spec into the issue.
3. If no issue was provided, synthesize the current conversation context, decisions, and testing seams into a formal spec (`to-spec` format), create the GitHub issue with `ready-for-agent`, and run the pipeline.
4. Execute in an isolated git worktree (`.worktrees/issue-<N>`), follow TDD (`skill://tdd`), run repo gates, self-review (`skill://code-review`), create PR (`skill://pr`), watch CI, squash-merge into `main`, sync the main branch, and tear down the worktree.
