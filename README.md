# rev-engine CLI

Run [RevEngine](https://github.com/laude-institute/rev-engine) locally, on a branch
or a pull request, as a third review pass beside Steward and CodeRabbit.

RevEngine ships as a ReviewBench agent container: it reads a pull request from a
fixed mount layout (`/work/repo`, `/work/pr`, `/work/out`) and writes a findings
file. This wrapper reproduces that contract on the local filesystem, so the
reviewer runs as a plain Node program — no Docker image, no GitHub Action, no
`OPENROUTER_API_KEY` if you use the Claude Code engine.

## Install

```sh
git clone https://github.com/keithah/rev-engine-cli.git ~/work/rev-engine-cli
git clone -b claude-code-engine https://github.com/clementperon/rev-engine.git ~/work/rev-engine
ln -sf ~/work/rev-engine-cli/bin/rev-engine ~/.local/bin/rev-engine
rev-engine doctor
```

Tested against `clementperon/rev-engine` at `b50f968` (branch `claude-code-engine`,
an unmerged third-party fork that adds the `claude-code` engine). With
`--engine api` the upstream `laude-institute/rev-engine` `main` works instead.

`doctor` checks Node >= 24, the engine checkout, the `claude` CLI and its login,
`gh`, and whether an OpenRouter key is present. Override locations with
`REV_ENGINE_HOME` and `REV_ENGINE_CACHE`.

RevEngine itself is Apache-2.0, copyright Laude Institute. This repository is
only the local runner; it contains none of their code.

## Use

```sh
# pre-PR: the committed branch against its merge-base (where Steward runs)
rev-engine branch --repo .

# post-PR: the PR head, in a throwaway worktree (where CodeRabbit runs)
rev-engine pr 147 --repo ~/work/yarr

# machine-readable, and usable as a gate
rev-engine branch --json --fail-on high

# show the inputs and settings without spending a review
rev-engine pr 147 --dry-run
```

Exit codes: `0` review produced, `1` reviewer failed, `2` usage or setup error,
`3` findings reached `--fail-on`.

### Modes

| Mode | Reviews | Tree |
|---|---|---|
| `branch` | `git diff <merge-base(base, HEAD)>..HEAD` | your working tree; must be clean |
| `pr <n>` | the PR diff at its head, via `gh` | a detached worktree under the cache; your checkout is untouched |

Both pass `RB_BASE` as the merge-base, so `read_base_file` sees what GitHub shows
as the base version and the diff is the same three-dot diff the PR page shows.

### Settings

| Flag | Container setting | Default |
|---|---|---|
| `--engine` | `RB_CONFIG_ENGINE` | `auto`: `claude-code` if `claude` is installed, else `api` |
| `--model` | `RB_CONFIG_MODEL` | `opus` (claude-code) / `openai/gpt-6-luna` (api) |
| `--verifier` | `RB_CONFIG_VERIFIER` | the reviewer model |
| `--verify` | `RB_CONFIG_VERIFY` | `lenient` |
| `--second-look` | `RB_CONFIG_SECOND_LOOK` | `on` |
| `--mode` | `RB_CONFIG_MODE` | `human` |
| `--min-severity` | `RB_CONFIG_MIN_SEVERITY` | `low` |
| `--max-findings` | `RB_CONFIG_MAX_FINDINGS` | 25 human / 80 agent |
| `--time-budget` | `RB_CONFIG_TIME_BUDGET` | 780s |
| `--max-turns` | `RB_CONFIG_MAX_TURNS` | 40 |
| `--effort` | `RB_CONFIG_EFFORT` | provider default |
| `--fit` | `RB_CONFIG_FIT` | off |
| `--set KEY=VALUE` | `RB_CONFIG_<KEY>` | — |

`--engine claude-code` runs each pass through `claude -p` against your local
Claude Code login, serving the reviewer's read-only tools back over a loopback
MCP server. It has no API key and reports cost at API prices in the log.
`--engine api` calls OpenRouter and needs `OPENROUTER_API_KEY`.

## Three passes

| Pass | Position | What it is good at |
|---|---|---|
| `steward` | pre-PR, local | policy/manifest form, repo rules, evidence that the branch is reviewable |
| `rev-engine branch` | pre-PR, local | independent Verify-then-Rank findings with line-accurate citations |
| CodeRabbit | post-PR, hosted | broad static analysis, walkthrough, per-line threads |
| `rev-engine pr <n>` | post-PR, local | a second opinion on the same diff, without touching your PR |

Both RevEngine modes are read-only and post nothing, so a third pass costs
nothing in PR noise. Run `branch` before opening the PR to fix what is cheap to
fix, and `pr` afterwards to catch what the other two missed — the interesting
comparison is findings neither Steward nor CodeRabbit raised.

The reviewer reads the project's rules from the base revision, so a branch cannot
rewrite the rules it is reviewed against. Findings, ranked details, the fit
verdict, what the reviewer read, and the reviewer's log are kept per run under
`~/.cache/rev-engine/runs/<timestamp>-<mode>-<repo>-<sha>/`.

## Limits

- **Prior decisions cover RevEngine's own threads only.** `--prior auto` collects
  what a maintainer decided on reviews RevEngine itself posted (it keys off the
  Action bot's marker), so CodeRabbit threads and bare human comments are not
  consulted. Pass `--prior path/to/prior.json` to supply decisions by hand, or
  `--prior off` to skip collection.
- **Nothing is posted to GitHub.** Use the Action for that.
- **`claude-code` drops the swarm and naive modes**, by design in this branch.
- **No benchmark scoring.** `scripts/score.sh`, `replay.ts` and `coverage.ts` need
  a ReviewBench clone and are unchanged here.
- **`branch` mode needs a clean tree**, because the reviewer reads the working
  tree at head; uncommitted edits would be invisible to it.

## Observed timings

Small 30-line diff, `--model haiku`: 17–25s, 6–7 findings.
A 373-line dependabot PR, `--model haiku`: 62s. Opus runs are slower and
pricier; the container enforces `--time-budget` and hands in what it has.
