---
name: pr-summary
description: "Summarise the current branch as a pull request description with risks, reviewer guidance, and a confidence score. Use when the user asks for a PR summary or description of a branch."
---

1. Pin the range: `git log --oneline <base>..HEAD` and `git diff --stat <base>...HEAD`. Use `origin/<default>` as the base when the local default branch is stale.
2. Read the diff itself, not just the commit messages; messages drift from what the code does.
3. Write the title: it often becomes the squash-commit message, so follow the repository's convention (recent merged PRs, `git log`) or `imperative-commits`. Name what changes for the user, not the mechanism: `Cut websocket frames by 70% with compression`, not `Negotiate permessage-deflate on the websocket`.
4. Write the summary:
   1. **Overview**: the problem in plain words (what was broken or missing, as the user met it), then the fix, in 1–3 sentences. Not a list of edits; those belong under key changes.
   2. **Key changes**: the main files and modifications, as a list.
   3. **Potential risks**: breaking changes, migrations, security concerns. Omit the section if there are none.
   4. **Review recommendations**: where the reviewer should look hardest, and why.
   5. **Confidence**: 1 (early work in progress) to 5 (production-ready), with one line justifying the score from what was tested.

Output the summary as Markdown ready to paste into the PR. Change nothing in the repo.
