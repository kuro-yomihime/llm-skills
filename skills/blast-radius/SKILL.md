---
name: blast-radius
description: "Find what a change could break outside its diff, and prove the one fact that makes it safe by running real code. Use before shipping or approving a small-looking change you don't fully trust, or when asked 'what could this break' or for the blast radius of a change."
metadata:
  adapted-from: "pstack blast-radius, Copyright (c) 2026 Lauren Tan, MIT License, https://github.com/cursor/plugins/tree/main/pstack"
---

Listing the callers is not the job; grep does that in seconds. The job is the breakage grep won't show. A writeup that sounds right proves nothing: it reads the same whether or not it is true. So find the one or two facts the change's safety rests on and prove them by running code.

## Evidence levels

For each fact the change's safety depends on, push it as far down this list as is cheap, and say where it stopped:

1. **Asserted.** Worth nothing on its own.
2. **Pointed at.** A real `file:line`, or the dependency's own source at the pinned version.
3. **Walked.** The failing case traced step by step and shown not to reach.
4. **Ran.** A script or test calls the real code and fails loudly if the fact is wrong.
5. **Reproduced** in the running app or service.

Level 4 is usually one small script that imports the same dependency version the project ships and calls the exact function in question, or that runs the real script or service in a sandbox (scratch `HOME`, temp dir, throwaway database).

## Steps

1. Pin the range: `git diff <base>...<head>`. For the current branch the head is `HEAD`, with `origin/<default>` as the base when the local default branch is stale; for one past commit, use `<sha>~1...<sha>`. Name what now behaves differently, including what the diff doesn't spell out: changed defaults, ordering, error paths, and code paths that become reachable for the first time.
2. Find the fact it is safe because of. Most risky-looking changes are safe because of a single fact, like "this call only drops cache entries that already expired". If it holds, most risks clear at once. Spend the time here, not on a long list of maybes.
3. Look where grep stops:
   - the dependency's source at the pinned version, plus any local patch or vendored copy
   - timing: startup and teardown order, async scheduling, retries, concurrent writers
   - data that outlives the process: serialized formats, API responses, DB columns, caches, queue messages, files another program reads
   - config and feature flags, and other services or repos that consume the same output
4. Rate each risk with a real likelihood and cost. Keep the confirmed ones; list checked-and-cleared ones separately. Cite a real `file:line`. A search that finds nothing is an answer: say what you searched. Never invent a caller or an API.
5. Prove the fact at level 4 or 5: write the script or test, run it, keep the command and its output.

Done when the safety fact is proven at level 4 or higher, or marked **unproven** with the level it reached, and every risk has a `file:line`.

## Report

- **Behaviour change**: what changed, including the non-obvious part.
- **Safe because**: the fact, its evidence level, and the proof (command and output), or **unproven**.
- **Risks**: how each breaks, `file:line`, likelihood and cost, the evidence level reached, how to check. Another bug your proof script surfaces goes here too.
- **Cleared**: what you checked and why it is fine.
- **Before merge**: the cheapest test or repro that would catch the real bug, including the script you wrote.

Keep scratch scripts in `/tmp`, not the repo, unless the proof should become a regression test. Strip secrets and internal hostnames before posting the report on a PR/MR.
