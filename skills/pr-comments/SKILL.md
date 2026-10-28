---
name: pr-comments
description: "Triage review comments on a GitHub pull request or GitLab merge request: judge each one, plan fixes for the valid ones, report the rest. Use when the user asks to address, handle, or respond to PR/MR comments or review feedback."
---

Every comment gets a verdict before any fix is planned. Reviewers are sometimes wrong; a suggestion applied blindly can break working code.

1. Identify the platform from `git remote get-url origin`: `github.com` is GitHub; `gitlab.com`, or any other host listed by `glab auth status`, is GitLab. For another unknown host, ask the user which platform it is.
2. Fetch the open review threads for the current branch. Skip resolved and outdated threads.
   - **GitHub**: use the GitHub MCP if available, otherwise the `gh` CLI:
     - `gh pr view --json number,url,reviews,comments`
     - inline review comments: `gh api repos/{owner}/{repo}/pulls/<number>/comments`
   - **GitLab** with `glab`:
     - `glab mr view --unresolved` lists the merge request's unresolved discussions
     - structured threads: take `iid` from `glab mr view --output json`, then `glab api "projects/:fullpath/merge_requests/<iid>/discussions" --paginate`. Keep discussions with a note where `resolvable` is true and `resolved` is false; inline notes carry `position.new_path` and `position.new_line`.
   - **GitLab** without `glab`: call the REST API with a token from `$GITLAB_TOKEN`, sending it as the `PRIVATE-TOKEN` header. `<project>` is the URL-encoded `group/repo` path (`/` becomes `%2F`):
     - find the MR: `GET https://<host>/api/v4/projects/<project>/merge_requests?source_branch=<branch>&state=opened`
     - threads: `GET https://<host>/api/v4/projects/<project>/merge_requests/<iid>/discussions?per_page=100`, following the `page` parameter until the result is empty
   If the API answers `401` (glab unauthenticated and no `$GITLAB_TOKEN`), stop and ask the user to run `glab auth login` or export `GITLAB_TOKEN`. Discussions require authentication even on public projects.
3. Establish intent before any verdict. Read the PR/MR description, linked issues, the complete diff, and the callers of the changed code. State the change's intent and its exclusions in 1–2 sentences; every verdict below is judged against that statement.
4. For each comment, read the code it points at and decide:
   - **valid**: the issue is real in the current code, and the change introduced or worsened it, or its stated behaviour requires the fix
   - **out of scope**: a real issue not introduced or worsened by this change and not required by its stated behaviour. Cite that boundary in one line and list it as a follow-up. Rarity, location in another file, or the proposed fix's shape is not sufficient evidence; a regression in a supported case stays valid. Impact sets the class, not the reviewer's label.
   - **invalid**: the suggestion is wrong, already handled, or conflicts with the task. Note the reason, with file:line evidence.
   - **unclear**: a question for the author or reviewer, including any comment whose verdict depends on uncertain product intent
5. For each valid comment, plan the fix: which file, what change, and how to verify it. Prefer the fix that deletes or narrows code over a new lock, cache, helper, or option; review rounds grow a PR one cheap-looking fix at a time.

Done when the intent statement is written and every unresolved comment has a verdict.

Report the intent statement first, then four groups (valid with planned fix, out of scope as follow-ups, invalid with reason, unclear with the question to ask), each item linking its comment.
Leave the code unchanged; the user applies the plan, e.g. with `/fix` or through another agent skill.
