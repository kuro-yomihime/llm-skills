---
name: pr-modernize
description: "Bring a stale GitHub PR / GitLab MR up to date and decide whether it should still exist (MODERNIZE, REWORK or CLOSE) before rebasing it. Use when the user asks to rebase, refresh, revive, or modernize an old PR/MR."
license: "MIT. Adapted from t3dotgg/tstack (Copyright (c) 2026 Theo Browne); see the repository LICENSE."
---

Adapted from the `modernize` skill of [t3dotgg/tstack](https://github.com/t3dotgg/tstack/blob/1085050f3baed1ac724caa4a8166145fb2ac88af/skills/modernize/SKILL.md) (MIT, Copyright (c) 2026 Theo Browne).

A clean rebase proves the commits still apply, not that the change still belongs in the product. Decide that first; rebase second.

## Guardrails

- Modernizing authorizes rebasing and pushing this PR's own head branch. It does not authorize merging, closing, retargeting the base, or widening scope.
- Existing worktree changes are the user's. If the worktree is dirty or in use, work in a disposable one (`git worktree add`).
- The target (base) repository and the head repository are separate. `origin` is often the author's fork, whose `<base>` branch can differ from the target's; never use `origin/<base>` as the base.
- Push only to the head repository's branch, with `--force-with-lease=refs/heads/<head-branch>:<head-sha>` (`<head-sha>` recorded in step 1). A rejected lease means someone else pushed: fetch and reassess, never fall back to `--force`.
- Post no PR/MR comment unless the user asked for one.

## 1. Establish intent

Identify the platform as in `pr-comments`. Resolve the PR/MR, then record its URL, state, author, CI status, review state, and four identities separately: **target repository + target branch**, **head repository + head branch**.

- **GitHub**: `gh pr view <n> -R <owner>/<repo> --json url,state,baseRefName,headRefName,headRepository,headRepositoryOwner,isCrossRepository,headRefOid,author,statusCheckRollup,reviewDecision`. The target repository is the `-R` repository the PR lives in; the head repository is `headRepositoryOwner.login`/`headRepository.name`.
- **GitLab**: `glab mr view <iid> --output json` (`target_project_id`, `target_branch`, `source_project_id`, `source_branch`, `sha`, `author`, `head_pipeline`). Resolve each project's clone URL with `glab api projects/<id>` (`ssh_url_to_repo` or `http_url_to_repo`).

Fetch both refs from their own repositories and pin them:

```sh
git fetch <target-url> refs/heads/<target-branch> && git rev-parse FETCH_HEAD   # <base-sha>
git fetch <head-url> refs/heads/<head-branch> && git rev-parse FETCH_HEAD       # <head-sha>
git merge-base <head-sha> <base-sha>                                            # <merge-base>
```

`<head-sha>` must equal the head SHA the platform reports; if not, refetch before going on. Use `<base-sha>`, never a branch name, in every later step, so a base that moves mid-task cannot change what you compared against.

Read the title, body, linked issues, commits, review threads, and the complete diff (`git diff <merge-base> <head-sha>`).

Done when you can state, in one sentence, the smallest coherent goal the PR was trying to reach, and the target and head identities, `<base-sha>`, `<head-sha>` and `<merge-base>` are recorded.

## 2. Check what changed around it

Read the base commits since the merge base (`git log --oneline <merge-base>..<base-sha>`), with the diff for the paths the PR touches. Search merged and open PRs/MRs for the same problem (`gh pr list -R <target> --state all --search <terms>`, `glab mr list -R <target> --all --search <terms>`).

Look past overlapping lines:

- Has the same user-facing problem already been solved?
- Did APIs, types, schemas, dependencies, or module boundaries change under it?
- Do the PR and the base now both claim the same behaviour or invariant?
- Was the code it changes deleted, moved, or deliberately simplified?
- Would landing it restore an old behaviour, or create a second way to do the same thing?
- Is the original benefit still worth the maintenance cost?

A rebase without conflicts says nothing about these questions.

Done when each question has an answer backed by a commit, a PR/MR link, or a file:line. An absence claim (e.g. "no replacement PR exists") is backed by the search queries you ran and their results.

## 3. Choose a disposition

- **MODERNIZE**: the goal is still useful and the implementation is still the right shape. Continue with step 4.
- **REWORK**: the goal is still useful, but a mechanical rebase would preserve the wrong implementation. Describe the smallest credible rewrite and stop, unless the user also asked for the rework.
- **CLOSE**: the PR is superseded, obsolete, or no longer worth its cost. Cite the evidence and stop. Do not rebase an obsolete branch; do not close it without explicit permission.

Age is not evidence, and sunk effort is not a reason to keep a PR alive.

## 4. Rebase (MODERNIZE only)

1. Check out `<head-sha>` in a disposable worktree; rebase it onto `<base-sha>` (`git rebase <base-sha>`).
2. Resolve each conflict by the PR's intent and the base's newer invariants. A conflict that reveals a product or architecture disagreement sends you back to step 3, not to a guess.
3. Compare the series with `git range-diff <merge-base>..<head-sha> <base-sha>..HEAD`, then compare the final target diff `git diff --stat <base-sha> HEAD` with the original `git diff --stat <merge-base> <head-sha>`. No intended PR behaviour disappeared; no already-landed change was reintroduced as a duplicate or conflicting implementation; no path or commit appears that the PR did not have before.
4. Run the project's checks for the touched area, and the change-specific verification. Separate failures the PR causes from ones already on the base or in the infrastructure.
5. **Stop without pushing** on an unexplained patch loss, a widened target diff, a failed change-specific verification, or an unresolved conceptual conflict. Record the concrete blocker with its evidence; a failure is not just another report field.
6. Push to the head repository and branch, then confirm the remote head equals the new local SHA:

   ```sh
   git push <head-url> HEAD:refs/heads/<head-branch> \
     --force-with-lease=refs/heads/<head-branch>:<head-sha>
   git ls-remote <head-url> refs/heads/<head-branch>
   ```

Never merge.

## Report

Lead with `MODERNIZE`, `REWORK`, or `CLOSE` and the one-line reason. Then:

- target and head identities, `<base-sha>` and `<merge-base>`;
- what changed on the base that matters to this PR;
- conceptual conflicts found, or that none were;
- the rebase and push result with the old and new head SHAs, or the blocker that stopped the push;
- checks run and their results;
- the decision or work left for the user, with the PR/MR link.

Separate what you observed from what you judge, and name what you could not verify.
