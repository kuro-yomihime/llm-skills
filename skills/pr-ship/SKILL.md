---
name: pr-ship
description: "Finish a branch: verify it is green, tidy the commit history, push, and open a pull request. Use when the user says ship, wrap up, or open a PR for finished work."
---

1. **Verify.** Run the project's tests, lint, and build. Stop and report if anything fails: a red branch does not ship.
2. **Tidy commits.** Organise the branch into a coherent history of logical, imperative-style commits (the `imperative-commits` skill). Rewrite history only on commits not yet pushed, unless the user agrees to a force-push.
3. **Push** the branch to its remote.
4. **Open the PR** with `gh`, `glab`, through API, or MCP. Write the body with the `pr-summary` skill. If the type of work warrants it, include screenshots to showcase changes in the top "Overview" section.
5. **Clean up** a worktree if one was created for this work, plus any temporary files the work left outside the repo.

Merge only when the user asks; shipping ends at an open PR.

Report the PR URL, what was verified, and anything skipped.
