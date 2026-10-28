---
name: pr-modernize
description: "Bring a stale GitHub PR / GitLab MR up to date and decide whether it should still exist: re-establish its intent, inspect what changed on the base since it diverged, choose MODERNIZE, REWORK or CLOSE, and rebase only when the idea still holds. Use when the user asks to rebase, refresh, revive, or modernize an old PR/MR."
---

A clean rebase proves the commits still apply, not that the change still belongs in the product. Decide that first; rebase second.

## Guardrails

- Modernizing authorizes rebasing and pushing this PR's own head branch. It does not authorize merging, closing, retargeting the base, or widening scope.
- Existing worktree changes are the user's. If the worktree is dirty or in use, work in a disposable one (`git worktree add`).
- Push a rebased branch only with `--force-with-lease=<branch>:<head SHA recorded in step 1>`. A rejected lease means someone else pushed: fetch and reassess, never fall back to `--force`.
- For a PR/MR from a fork, push to the head repository's branch, never to upstream.
- Post no PR/MR comment unless the user asked for one.

## 1. Establish intent

Identify the platform as in `pr-comments`. Resolve the PR/MR and record its URL, state, base branch, head repository and branch, head SHA, author, CI status, and review state:

- **GitHub**: `gh pr view <n> --json url,state,baseRefName,headRefName,headRepositoryOwner,headRefOid,author,statusCheckRollup,reviewDecision`
- **GitLab**: `glab mr view <iid> --output json` (`target_branch`, `source_branch`, `source_project_id`, `sha`, `author`, `head_pipeline`)

Read the title, body, linked issues, commits, review threads, and the complete diff. Fetch the base and the head, and record `git merge-base <head> origin/<base>`; it separates what the PR changed from what the base changed since.

Done when you can state, in one sentence, the smallest coherent goal the PR was trying to reach, and the head SHA and merge base are recorded.

## 2. Check what changed around it

Read the base-branch commits since the merge base (`git log --oneline <merge-base>..origin/<base>`), with the diff for the paths the PR touches. Search merged and open PRs/MRs for the same problem (`gh pr list --state all --search <terms>`, `glab mr list --all --search <terms>`).

Look past overlapping lines:

- Has the same user-facing problem already been solved?
- Did APIs, types, schemas, dependencies, or module boundaries change under it?
- Do the PR and the base now both claim the same behaviour or invariant?
- Was the code it changes deleted, moved, or deliberately simplified?
- Would landing it restore an old behaviour, or create a second way to do the same thing?
- Is the original benefit still worth the maintenance cost?

A rebase without conflicts says nothing about these questions.

Done when each question has an answer backed by a commit, a PR/MR link, or a file:line.

## 3. Choose a disposition

- **MODERNIZE**: the goal is still useful and the implementation is still the right shape. Continue with step 4.
- **REWORK**: the goal is still useful, but a mechanical rebase would preserve the wrong implementation. Describe the smallest credible rewrite and stop, unless the user also asked for the rework.
- **CLOSE**: the PR is superseded, obsolete, or no longer worth its cost. Cite the evidence and stop. Do not rebase an obsolete branch; do not close it without explicit permission.

Age is not evidence, and sunk effort is not a reason to keep a PR alive.

## 4. Rebase (MODERNIZE only)

1. Branch or worktree at the recorded head SHA; rebase it onto the freshly fetched `origin/<base>`.
2. Resolve each conflict by the PR's intent and the base's newer invariants. A conflict that reveals a product or architecture disagreement sends you back to step 3, not to a guess.
3. Compare the series before and after with `git range-diff <merge-base>..<old head> origin/<base>..HEAD`, then read the final diff. No intended behaviour may disappear; no behaviour the base already has may come back.
4. Run the project's checks for the touched area. Separate failures the PR causes from ones already on the base or in the infrastructure.
5. Push with the lease from the guardrails, then confirm the remote head equals the new local SHA (`git ls-remote <remote> refs/heads/<branch>`).

Never merge.

## Report

Lead with `MODERNIZE`, `REWORK`, or `CLOSE` and the one-line reason. Then:

- what changed on the base that matters to this PR;
- conceptual conflicts found, or that none were;
- the rebase and push result with the old and new head SHAs, if performed;
- checks run and their results;
- the decision or work left for the user, with the PR/MR link.

Separate what you observed from what you judge, and name what you could not verify.
