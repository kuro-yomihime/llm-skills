# Fleet constants: ar-3

The only file holding fleet-specific values. `dispatch`, `review-fix-loop`, `burn-window` and `fleet-hygiene-pass` use placeholders and point here. Update this file when models, paths or cadence change.

| Placeholder | Value |
|---|---|
| `<HOST>` | `ar-3` |
| `<OPERATOR>` | Master |
| `<IMPLEMENTER_MODEL>` | `anthropic/claude-opus-5-5` |
| `<REVIEWER_MODEL>` | `openai-codex/gpt-6.1-sol` |
| `<MENU>` | `~/reports/gpt-burner/menu.md` |
| `<RECORDS>` | `~/reports/dispatch/` |
| `<RUN_LOGS>` | `~/reports/gpt-burner/*.log` |
| Shell prompt | `[arch@p1g3-ar-3 ...]$` |
| `<WINDOW_LOCK>` | `~/reports/gpt-burner/window.lock` |
| Window cadence | gpt-menu-runner, about every 5h |
| Concurrency cap | 3 model sessions on the host in total (OOM history below); `fleet-hygiene-pass` staggers dispatches against it |
| Signature | `<model>, <harness>`, e.g. `gpt-6.1-sol, omp` |
| Collector | OpenClaw automations job (`pr-babysit` `references/openclaw.md`) |
| Harness | omp; `@file`, `--max-time`, `--session-dir` and the `model_change` record verified on omp 18.12.6; re-check `omp --version` on the host after upgrades |

Provider prefixes must match `omp models` on the host; `--model` fuzzy-matches anything shorter.

## Quota facts

- `<REVIEWER_MODEL>`: rolling 5-hour window plus a weekly cap.
- `<IMPLEMENTER_MODEL>`: separate consumption limits; it walls too, so probe it every window.

## Host hygiene

- Prefer local tools for media (piper TTS, ffmpeg) over cloud keys.
- Secrets arrive as files with `chmod 600`; never echoed; rotated after any chat exposure.

## Incident log

| Incident | Owning rule |
|---|---|
| Heredoc and nested-quote prompts crashed launches | `dispatch` Step 3 (`@file`) |
| Threads silently ran on a fallback model | `dispatch` Step 4 |
| Threads pushed to forks while MRs tracked the upstream branch | `dispatch` Step 1 push target, Step 6 |
| `DONE:` claimed, nothing pushed; verdicts never posted | `dispatch` Step 6 |
| Duplicate footer links added | `dispatch` Step 1 inspect-before-adding |
| Collector fired after a scrub and reported a corpse | `dispatch` Step 8 order |
| Probes hung for 9 hours | `dispatch` Step 4 probe, `--max-time` |
| Menu declared exhausted while NEEDS CHANGES verdicts stood | `burn-window` Step 3 reconcile |
| Reviews skipped silently on quota walls | `review-fix-loop` quota rule |
| Oct 6: three concurrent model sessions caused OOM, SIGTERM on running work | concurrency cap above |
