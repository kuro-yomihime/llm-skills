---
name: pr-ship
description: "Finish a branch: verify it is green, tidy the commit history, push, and open a pull request. Use when the user says ship, wrap up, or open a PR for finished work."
---

1. **Verify.** Run the project's tests, lint, and build. Stop and report if anything fails: a red branch does not ship.
2. **Prove.** Green checks show nothing broke, not that the change works. For each change, get evidence you observed in this session:
   - **Bug fix**: run the same focused reproduction against the pre-fix revision in a disposable worktree (`git worktree add <tmp> <pre-fix commit>`; `HEAD` when the fix is still uncommitted) and against the final head. Never revert, stash, reset, or check out over the user's work to reconstruct history.
   - **UI or runtime behaviour**: run the app and look; mocked tests do not prove the user experience.
   - **Proof unavailable** (no pre-fix revision, runtime, or environment): name the exact limitation and report the change as unproven, not implied to work.
   - **Proof fails**: the changed behaviour demonstrably fails on the final head. Stop before pushing and report it like a failing check.
3. **Tidy commits.** Organise the branch into a coherent history of logical, imperative-style commits (the `imperative-commits` skill). Rewrite history only on commits not yet pushed, unless the user agrees to a force-push.
4. **Push** the branch to its remote.
5. **Find an existing PR/MR** for this branch (`gh pr list --head <branch>`, `glab mr list --source-branch <branch>`). One exists → update it; never open a second.
6. **Compose** the title and body with the `pr-summary` skill, plus one line per change on how it was proved, or its named limitation. If the type of work warrants it, include screenshots in the Overview section.
7. **Check privacy, then publish.** Check the title, body, and attachments for secrets and private details, then open or update the PR/MR with `gh`, `glab`, the API, or MCP.
8. **Clean up** worktrees created for this work, including proof worktrees, plus any temporary files the work left outside the repo.

Merge only when the user asks; shipping ends at an open PR. Shipping grants no merge or extra testing permissions.

Report the PR URL, what was verified, how each change was proved, and anything unproven or skipped.
