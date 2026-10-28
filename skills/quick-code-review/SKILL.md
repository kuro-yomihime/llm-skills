---
name: quick-code-review
description: "Fast read-only review of uncommitted working-tree changes, with findings ranked by priority. Use when the user asks for a quick review of current or local changes before committing."
---

Review what is in the working tree right now, not a branch or PR. Report findings; leave the code unchanged.

1. Collect the changes: `git diff` for unstaged edits, and `git status --short` for untracked files (read those in full). If nothing is unstaged but changes are staged, review `git diff --staged` instead and say so.
2. Read each changed file around the edited hunks, enough to judge them in context.
3. Check:
   - correctness
   - code is clear; names say what they hold
   - no duplicated code
   - errors handled; input validated at boundaries
   - no secrets or keys
   - tests cover the changed behaviour: each would fail if the observable behaviour it asserts broke, and keep passing if only the internals changed
   - performance on hot paths
   - bugs and edge cases
   - guards, retries, clamps, swallowed errors: is the state they handle legitimate? If not, they hide a bug

Done when every changed file has been read.

Report findings by priority, each with file:line and a concrete fix (with a code example where it helps):

- **Critical**: must fix
- **Warning**: should fix
- **Suggestion**: worth considering

Get straight to the findings. If there are none, say so in one line.
