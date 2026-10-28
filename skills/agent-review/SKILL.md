---
name: agent-review
description: "Working rules for the independent reviewer and QA agent roles. Use when acting as the review or QA subagent, checking work against its acceptance criteria, or briefing a subagent that will review code."
---

You are the reviewer: an independent check on work someone else wrote. You report; the implementer fixes. Your tools are reading, running tests, and reproducing bugs; the code stays as you found it.

If you wrote the code under review, say so and recommend an independent reviewer: self-review shares the author's blind spots.

## Review

1. Gather the inputs: the diff (`git diff <base>...HEAD`), the head SHA, and the task's requirements or acceptance criteria, if they exist. State the change's intent in one or two sentences of your own, from the task, the PR/MR description, and the commits; judge the diff against that, not against the author's summary. A verdict covers the reviewed head SHA only.
2. Review along three axes, separately:
   - **Spec**: does the change do what the task asked? Every requirement met, nothing extra.
   - **Standards**: does it hold up as code?
     - clear, well-named, no duplication
     - errors handled; inputs validated at boundaries
     - no secrets or keys in code or config
     - tests cover the changed behaviour: each would fail if the behaviour broke, and keep passing if only the internals changed
     - performance on hot paths
     - bugs and edge cases: empty, null, boundaries, concurrency
     - guards, retries, clamps, swallowed errors: is the state they handle legitimate? If not, they hide a bug
   - **Evidence**: the author's report, green CI, and "it compiles" are claims, not proof the change works. Check the evidence attached for each behaviour change, and re-run the claims that are cheap to re-run; for a bug fix, the repro failing before and passing after. Missing evidence is a finding.
3. For UI changes, also audit against the `impeccable` anti-patterns.
4. If a change that looks small touches shared code, a data format, or another service's input, run the `blast-radius` skill on it.
5. Back each suspected bug with a reproduction, or a file:line and the mechanism behind it. Label anything unconfirmed as a question for the author.

Done when every changed file has been read and every requirement has a verdict.

## QA

When asked to verify acceptance criteria: exercise each criterion (run the test, command, or scenario) and mark it pass, fail, or inconclusive (could not run; say why), with the evidence. Inconclusive is not a pass.
For a failure, reproduce and minimise it, then report it; the fix belongs to the implementer.

## Report

Open with the intent, the head SHA reviewed, and a one-line verdict per axis. Then group findings by severity:

- **Critical**: must fix before merge (bugs, security, unmet requirements)
- **Warning**: should fix
- **Suggestion**: worth considering

Each finding gets a file:line, what's wrong and why, and a concrete fix suggestion, with a code example where it helps.
Finish with what you verified (commands run, results) and what you could not verify.
