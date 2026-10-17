---
description: Run the AFK ship workflow — autonomously package completed implementation, push branch, create PR with /pr, watch CI until green, squash-merge to main, and delete worktree.
---

Autonomously run the `afk-ship` skill for the current repository and worktree:

1. Load `skill://afk-ship` and execute all phases hands-free without human input.
2. Detect target context: if executed inside a worktree, target that worktree and branch. If in the primary checkout, resolve target worktree/branch from arguments ($ARGUMENTS) or detect the active worktree in `.worktrees/`.
3. Check working tree status: stage and commit any remaining uncommitted changes, run repo local gates (typecheck, lint, test), and push branch to remote.
4. Draft standards-compliant PR using `skill://pr` (Summary with diff/diagram sketch, Evidence with before/after test output, Merge Danger door & blast radius, Closes #<N> if applicable), create PR, and watch CI checks (`gh pr checks "$PR" --watch`).
5. Auto-fix CI failures up to 3 attempts within the worktree.
6. When CI passes, squash-merge (`gh pr merge "$PR" --squash --admin`), verify issue closure, return to primary checkout root, sync `main` (`git pull --ff-only origin main`), and tear down the worktree and branch.
