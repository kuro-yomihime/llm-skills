---
name: "review-fix-loop"
description: "Implementer-driven review loop: after pushing, spawn a separate reviewer session on a different model, fix or dispute its findings, and repeat until the MR verdict is MERGE CLEAN (max 5 rounds). Use when you are the implementer of a large change and were told to run the review loop yourself."
---

# Review-fix loop

**Layer:** `burn-window` → `dispatch` → **`review-fix-loop`** (inside the implementer thread).

The implementer drives the loop; the review is independent. This skill owns review rounds. Launching, model verification and probing belong to the `dispatch` skill (Steps 3–4); fleet values (`<REVIEWER_MODEL>`, signature) to its `references/ar-3.md`.

## Rules

- **Independence**: the reviewer runs in a separate session **and** on a different model family from the implementer. Never review your own work.
- **Verdicts bind to a head.** A verdict counts only if its first line names the SHA that equals the MR API `sha`.
- **Hard cap: 5 rounds.** Round N = the number of verdict notes on this MR (first line `**Review verdict:`) plus one, so a restarted implementer resumes the count.
- **Quota**: reviewer probe or round hits a usage limit → stop and report `BLOCKED (reviewer quota, reset <time|unknown>)` immediately. Never wait it out, skip the review, or review it yourself; the next `burn-window` picks it up as a re-review.
- **Never merge.** MERGE CLEAN ends the loop.

## The loop

1. **Push**: logical imperative commits; the project's lint, build and test commands green; push to the MR source branch; verify per `dispatch` Step 6 (`kind: implement`).
2. **Spawn the reviewer**: fill the template below, launch session `<task>-review-<N>` on `<REVIEWER_MODEL>` per `dispatch` Steps 3–4, with `--max-time 60m`; its dispatch record has `kind: review`.
3. **Poll**: every 120s check the log for `DONE:` (`sleep 120; grep -q '^DONE:' /tmp/<task>-review-<N>.log`). Session gone without `DONE:`, or `--max-time` passed → kill, retry the round once; second failure → report `BLOCKED (reviewer hung)`.
4. **Read the verdict** from the notes API (`glab api projects/<id>/merge_requests/<IID>/notes?sort=desc&order_by=created_at`). Check that the head in its first line equals the MR API `sha`; otherwise it is void, rerun the round.
5. **NEEDS CHANGES** → triage every item, like the `pr-comments` skill:
   - valid → fix exactly that item;
   - wrong → reply on the MR with file:line evidence instead of fixing.
   Gates green, push, verify the new head, go to step 2 with N+1. The next reviewer prompt lists your disputes.
6. **MERGE CLEAN** → stop and report.

## Reviewer template

Fill every placeholder; leave none.

```
Review MR !<IID> on <REPO_URL>: <BRANCH> into <TARGET>, head <SHA>. Round <N>/5.
Follow the agent-review skill. Clone or fetch, review the full diff
origin/<TARGET>...origin/<BRANCH> as a hostile senior engineer: correctness,
security, authz, injection, data exposure, test coverage. Run the project's
lint, build and test commands yourself.
Disputes from the implementer on earlier rounds (accept each or re-raise it with evidence):
<DISPUTES|none>
Any Critical finding → NEEDS CHANGES; otherwise MERGE CLEAN.
Post exactly one MR note. First line:
**Review verdict: <MERGE CLEAN|NEEDS CHANGES>** — head `<SHA>` — round <N>/5
Then numbered findings with file:line and severity. Last line: _<model>, <harness> — dispatched by Dalian on <date>_
Never push, approve or merge. Final line of your output starts with DONE: and
includes the note URL.
```

## Report

Rounds run, the verdict note URL for each round, final head SHA, disputed items and their outcome, and the `BLOCKED` reason if any.

## Failure modes

| Incident | Rule |
|---|---|
| "Verdict landed" claimed, nothing posted | step 4 reads the notes API |
| Review of a stale head | head SHA in the verdict's first line |
| Wrong finding "fixed" or looped to the cap | disputes with evidence |
| Reviewer skipped under quota pressure | quota rule: BLOCKED, never skipped |
| Endless fix rounds | 5-round cap counted from MR notes |
