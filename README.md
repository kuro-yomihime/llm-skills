# llm-skills

LLM [Agent Skills](https://agentskills.io) consumed by multiple LLM harnesses: **omp**, **opencode**, **openclaw**, **hermes**, etc.

## Install

```sh
make install    # or: ./tools/install.sh
```

Idempotent; re-run after every `git pull`. It:

1. lints every skill (aborts on problems),
2. symlinks `skills/<name>` into the shared hub `~/.agents/skills`, pruning links for removed skills,
3. wires harnesses that don't read the hub natively (hermes: adds the hub to `skills.external_dirs` in `~/.hermes/config.yaml`).

omp, opencode and openclaw read `~/.agents/skills` on their own.

Override the hub with `SKILLS_HUB=/path ./tools/install.sh`.

Other targets: `make uninstall`, `make lint`, `make link` (link + prune without full install).

## The skills

| Skill | Use when |
|---|---|
| `agent-implement` | Acting as the implementation subagent; executing a plan step by step |
| `agent-review` | Acting as the independent reviewer/QA subagent |
| `blast-radius` | Finding what a change could break outside its diff, and proving the fact it is safe because of |
| `imperative-commits` | Committing changes, splitting work into commits, writing commit messages |
| `pr-comments` | Triaging review comments on a GitHub PR / GitLab MR |
| `pr-babysit` | Watching one PR/MR in the background until merge (hermes, openclaw, local timer, remote host) |
| `pr-ship` | Finishing a branch: verify green, tidy history, push, open PR |
| `pr-summary` | Writing a PR summary/description for the current branch |
| `quick-code-review` | Fast read-only review of uncommitted working-tree changes |
| `quick-fix` | Fixing issues identified earlier in the conversation (user-invoked only) |
| `visual-code-review` | Rendering a PR/branch review as a step-by-step HTML walkthrough |

## External skills

Some external skills that I highly recommend:

- [pbakaus/impeccable](https://github.com/pbakaus/impeccable)
- [mattpocock/skills](https://github.com/mattpocock/skills)
- [coreyhaines31/marketingskills](https://github.com/coreyhaines31/marketingskills)
- [life-alignment-agent](https://clawhub.ai/lakendocean/skills/life-alignment-agent)

And some others that seem interesting:

- [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd)
- [blader/humanizer](https://github.com/blader/humanizer)

## License

[MIT](LICENSE)
