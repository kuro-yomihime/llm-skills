---
name: pr-ship
description: "Finish a branch: verify it is green, tidy the commit history, push, and open a pull request. Use when the user says ship, wrap up, or open a PR for finished work."
---

1. **Verify.** Run the project's tests, lint, and build. Stop and report if anything fails: a red branch does not ship.
2. **Prove.** Green checks show nothing broke, not that the change works. For each change, get evidence you observed in this session: for a bug fix, reproduce the failure, apply the fix, and re-run the same reproduction; for UI or runtime behaviour, run the app and look (mocked tests do not prove the user experience). A change you could not prove is reported as unproven, not implied to work.
3. **Tidy commits.** Organise the branch into a coherent history of logical, imperative-style commits (the `imperative-commits` skill). Rewrite history only on commits not yet pushed, unless the user agrees to a force-push.
4. **Push** the branch to its remote.
5. **Open the PR** with `gh`, `glab`, through API, or MCP. First check that none exists for this branch (`gh pr list --head <branch>`, `glab mr list --source-branch <branch>`); update an existing one instead of opening a second. Write the body with the `pr-summary` skill, plus one line on how each change was proved. If the type of work warrants it, include screenshots to showcase changes in the top "Overview" section. Before submitting, check the title, body, and attachments for secrets and private details.
6. **Clean up** a worktree if one was created for this work, plus any temporary files the work left outside the repo.

Merge only when the user asks; shipping ends at an open PR.

Report the PR URL, what was verified, how each change was proved, and anything unproven or skipped.
