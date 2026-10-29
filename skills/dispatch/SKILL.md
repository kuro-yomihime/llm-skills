---
name: "dispatch"
description: "Launch one headless LLM agent thread on a remote host and confirm delivery by evidence: write the brief as a file, launch in tmux, verify the resolved model, arm a one-shot collector, scrub on kill. Use when handing an implement/review/fix/research task to a background agent thread, when probing whether a model has quota, or when killing a thread."
---

# Dispatch

**Layer:** `burn-window` → **`dispatch`** → `review-fix-loop` (inside the implementer thread).

One thread, one tmux session, one dispatch record, one collector. This skill owns launch, model verification, probing, collection and scrub; `burn-window` and `review-fix-loop` point here instead of restating them.

Placeholders: `<HOST>`, `<IMPLEMENTER_MODEL>`, `<REVIEWER_MODEL>`, `<OPERATOR>`, `<RECORDS>` (dispatch record dir). Fleet values and the incident log: [references/ar-3.md](references/ar-3.md). Always pass models as full `provider/id`; `--model` is a fuzzy match, which is how silent substitution happens.

**SHA and API state are facts; agent reports are hypotheses.** A `DONE:` line is a claim until Step 6 confirms it.

## Step 1: Write the brief

Write it to a local file, `chmod 600`, `scp` to `<HOST>:/tmp/prompt-<session>.txt`. Briefs never contain tokens; the thread uses the host's existing `glab`/`gh` auth. A brief contains, in order:

1. **Grounding**: workdir/clone path; "read README.md and docs/ first"; facts it must not re-derive (current heads, merged MRs, prior verdicts), marked "verify, do not trust this summary".
2. **Deliverables**: files, branch, imperative commits, the project's lint, build and test commands green. Large tasks add: "after the first push, run the `review-fix-loop` skill".
3. **Guardrails**:
   - Push target: `<exact remote URL>` `<branch>`, the MR's `source_project` and `source_branch`. Pushing anywhere else counts as failure.
   - Never push the default branch; never merge; never print secrets.
   - Inspect before adding: a link, file or line must not already exist.
   - Time budget: `<N>m`, matching `--max-time`.
4. **Closure**: "If blocked, stop and print `BLOCKED: <evidence>` as the final line; do not improvise scope. Otherwise print `<specifics>`; final line starts with `DONE:`."

Vague intent from `<OPERATOR>` loosens *Deliverables* only. Grounding, Guardrails and Closure are mandatory.

## Step 2: Record the baseline

Write `<RECORDS>/<session>.json` atomically (temp file + rename): session, task, `kind` (`implement` or `review`; selects the Step 6 evidence), model, host, workdir, push URL, branch, MR IID, baseline `git ls-remote <push URL> refs/heads/<branch>` SHA (or `absent`), prompt path, log path, collector ID (filled in Step 5), artifacts created (appended as they appear). This record is the only memory a restarted operator has: after a host reboot, list `<RECORDS>`, compare against `tmux ls` and `ls-remote`, and finish or scrub each orphan.

## Step 3: Launch

```sh
tmux new-session -d -s <session> -c <workdir> \
  "omp -p --model <provider>/<model-id> --max-time <N>m --session-dir /tmp/omp-sess/<session> @/tmp/prompt-<session>.txt 2>&1 | tee /tmp/<session>.log; exec bash"
```

- `@file` passes the brief without `$(cat …)` or nested quoting.
- `--max-time` is the hard ceiling against hung threads.
- `tee` keeps the `DONE:` line after the pane scrolls.
- Corrections to a running thread: write a new prompt file and launch a follow-up the same way; never type prompts through `send-keys`.

## Step 4: Verify the resolved model

Done when the first `model_change` record in `/tmp/omp-sess/<session>/*.jsonl` has `model == <provider>/<model-id>` **and** `resolvedModelIsFallback == false`:

```sh
jq -c 'select(.type=="model_change") | [.model, .resolvedModelIsFallback]' /tmp/omp-sess/<session>/*.jsonl | head -1
```

Never grep for the model name: briefs mention model names. Missing record or mismatch → kill, relaunch; silent fallback is a defect.

**Probe** (quota check before dispatching to a model): `timeout 150 omp -p --model <provider>/<id> --max-time 2m --session-dir /tmp/omp-sess/probe-<id>-<run>/ "Reply with OK"`, where `<run>` is fresh per invocation (`$(date +%s)-$$`); never reuse a probe session directory. Live = output contains `OK` **and** the Step 4 check passes against that invocation's directory only. A fallback answer is not live. Usage-limit error → record the reset time if given, else `retry: next window`.

## Step 5: Arm the collector

A scheduled check that reports exactly once, then disables itself. Use `pr-babysit`'s scheduler mechanics for your harness (its `references/`). Record the job ID in the dispatch record. Fill every placeholder:

```
You are collector for <session> on <HOST>. Log: /tmp/<session>.log
Kind: <implement|review>. MR !<IID>. Implement: expected push <push URL> <branch>, baseline <BASELINE_SHA|absent>. Review: expected verdict note, no push.
Each ssh check finishes in ≤5s; never attach to or type into the session.
Run once; increment `collector_firings` in <RECORDS>/<session>.json first.
- No DONE:/BLOCKED: line, session alive, firing < <K> (default 3) → reply NO_REPLY.
- No such line, session alive, firing <K> → reply once "<task>: STALLED <last log line>"; disable job <JOB_ID>.
- No such line, session gone → reply "<task>: FAILED: <last log line>"; disable job <JOB_ID>.
- DONE: → verify per the dispatch skill Step 6, then reply one line
  "<task>: <sha> <verdict note URL|push URL>" or "<task>: FAILED: <evidence>"; disable job <JOB_ID>.
- BLOCKED: → reply "<task>: BLOCKED: <evidence>"; disable job <JOB_ID>.
```

Timing: first firing at 15–30 min for simple tasks, 40+ min for builds or web research. Collector unavailable in this harness → check inline at the next opportunity.

## Step 6: Verify delivery

Evidence depends on the record's `kind`.

**`implement`:**

1. `git ls-remote <MR source project URL> refs/heads/<branch>` returns a SHA that differs from the baseline.
2. MR API `sha` equals that SHA (a push to any other remote leaves it unchanged).
3. Claimed comments exist in the notes API; claimed deletions are absent (`ls`, `ls-remote`).

**`review`** (reviewers never push; an unchanged head is expected):

1. The verdict note exists in the notes API; record its URL.
2. The SHA the note names as reviewed equals the current MR API `sha`.

Any check fails → report `FAILED: <evidence>`, never the thread's claim.

## Step 7: Review hand-off

Large tasks: the implementer runs the `review-fix-loop` skill itself (its brief says so, Step 1). Small tasks: dispatch one reviewer thread on `<REVIEWER_MODEL>` with the `review-fix-loop` reviewer template, one round, recorded as `kind: review`.

## Step 8: Scrub

When a thread is killed, in this order:

1. Cancel the collector (the only part that can fire mid-scrub).
2. `tmux kill-session -t <session>`.
3. Delete only artifacts listed in the dispatch record: pushed branch (`glab api -X DELETE projects/<id>/repository/branches/<branch>` if `git push --delete` fails), prompt file, log; close the MR if the thread opened one.
4. Done when `tmux has-session -t <session>` fails, `ls-remote` is empty, and the collector reads back as cancelled. Mark the record `scrubbed`.

Finished threads: reap the session and delete the prompt file once delivery is verified.

## Failure modes

| Incident | Rule | Check |
|---|---|---|
| Nested quoting crashed launches | Step 3 `@file` | session alive, log non-empty |
| Silent model fallback | Step 4 | `model_change` record |
| Pushes to a fork while the MR tracked upstream | Step 1 push target | Step 6.2 MR `sha` |
| Unverified `DONE:` claims; verdicts never posted | Step 6 | `ls-remote`, notes API |
| Duplicate footer links | Step 1 inspect-before-adding | reviewer round |
| Collector delivered a scrubbed corpse | Step 8 order | collector cancelled |
| 9-hour hung probes | `--max-time`, `timeout 150` | process exits |
