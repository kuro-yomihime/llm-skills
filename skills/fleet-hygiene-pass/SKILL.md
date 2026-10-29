---
name: "fleet-hygiene-pass"
description: "Run a hygiene pass on a dispatch host: use when tmux sessions accumulate, memory runs tight, /tmp debris builds up, or after heavy dispatch rounds"
---

# Fleet Hygiene Pass

How to clean `<HOST>`: reap finished tmux sessions, watch memory against the fleet concurrency cap, and clear transient debris, without touching live work or foreign sessions. Run after heavy dispatch rounds, when sessions accumulate, or when memory runs tight.

Placeholders and fleet values (`<HOST>`, `<OPERATOR>`, `<RECORDS>`, `<RUN_LOGS>`, shell prompt, concurrency cap, OOM history): the `dispatch` skill's `references/ar-3.md`. Completion, delivery verification and cleanup belong to the `dispatch` skill (Steps 6 and 8); this skill only decides which sessions to hand to it.

## Step 1: Inventory and classify every session

`tmux ls`, then match each session to its dispatch record in `<RECORDS>`. Classify:

- **VERIFIED** — the dispatch record shows delivery verified per `dispatch` Step 6. Reap candidates. A `DONE:` line or a bare shell prompt alone is a claim, not verification; run `dispatch` Step 6 first and classify by its result.
- **RUNNING** — no verified delivery and the thread is still working. Never touch.
- **FAILED** — `dispatch` Step 6 failed or the thread exited without `DONE:`/`BLOCKED:`. Report to `<OPERATOR>`; scrub only per `dispatch` Step 8.
- **FOREIGN** — attached sessions, `<OPERATOR>`'s own work, anything without a dispatch record. Never touch, even if idle.

## Step 2: Reap the verified, one by one

Follow `dispatch` Step 8 "Finished threads" for each VERIFIED session, one `tmux kill-session -t <session>` per call (a single kill with multiple `-t` flags silently no-ops). Survivors are only RUNNING, FAILED and FOREIGN sessions.

## Step 3: Check memory against the concurrency cap

`free -m`. If used memory is high while several model threads are active, reap VERIFIED sessions first (they hold memory too) and stagger heavy dispatches; never exceed the fleet concurrency cap.

## Step 4: Clear transient debris

- `/tmp/prompt-*.txt` and logs only for sessions whose dispatch record is verified or scrubbed; one-off scratch clones in `/tmp`
- Run logs under `<RUN_LOGS>` when they accumulate: keep the menu, triage reports and review verdicts; the raw run-chatter goes
- Delivered bundles, stale screenshots, superseded working copies

Keep: every report, decision record, dispatch record and live work product. If unsure whether a file is debris, leave it; hygiene passes recur.

## Step 5: Report tersely

What was reaped (list), survivors (with why: running/failed/foreign), memory state. `<OPERATOR>` wants the ledger, not the process.

## Failure modes this prevents

Shell-sprawl after busy rounds; OOM killing live work mid-gate; /tmp filling with dispatch corpses; accidental kills of foreign sessions; reaping a thread whose `DONE:` was never verified.
