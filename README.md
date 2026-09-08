![Codex Fan-Out](assets/codex-fanout-banner.png)

# Codex Fan-Out

![Codex CLI skill](https://img.shields.io/badge/Codex_CLI_skill-000000?style=for-the-badge)
![Git worktrees](https://img.shields.io/badge/Git_worktrees-F05032?style=for-the-badge&logo=git&logoColor=white)
![MIT licence](https://img.shields.io/badge/MIT_licence-blue?style=for-the-badge)

**Fan work out to headless Codex CLI workers, up to six at once.**

> The worker does the bulk, you keep the judgement.

_Fan-out_ is dispatching many independent workers at once and reviewing what comes back.

## Table of Contents

<details>
  <summary>Expand</summary>
  <ol>
    <li><a href="#what-it-does">What It Does</a></li>
    <li><a href="#quick-start">Quick Start</a></li>
    <li><a href="#which-model-to-use">Which Model to Use</a></li>
    <li><a href="#how-it-stays-safe">How It Stays Safe</a></li>
    <li><a href="#what-it-cannot-do">What It Cannot Do</a></li>
    <li><a href="#under-the-hood">Under the Hood</a></li>
    <li><a href="#go-deeper">Go Deeper</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#licence">Licence</a></li>
  </ol>
</details>

## What It Does

- **Headless Codex CLI workers.** Each worker is `codex exec` in a dedicated Git worktree, using the account you already authenticated.
- **You Write the Brief, It Writes the Files, You Review.** The worker reads the repository's `AGENTS.md` on its own, so the brief carries the task and not the house rules.
- **Up to Six at Once.** Launch them in parallel, each with its own brief, log, error file, report, and worktree, and never let two workers write the same file.
- **A Real Harness.** It gives you a real exit code and JSONL output, and it reads `AGENTS.md` on its own.

If a task needs your judgement or your conversation context, you do not need this skill.

## Quick Start

1. **Check the Prerequisites.** You need Codex CLI 0.153.4 or newer, Git, `curl`, and `jq`.

1. **Clone Into the Global Skills Directory**, then restart Codex so the skill loads. It is available as `$codex-fanout` from then on.

   ```bash
   git clone https://github.com/TolongLabs/codex-fanout "${CODEX_HOME:-$HOME/.codex}/skills/codex-fanout"
   ```

1. **Choose a Model.** Because the worker runs with `--ignore-user-config --ignore-rules`, it does not load your default model from `config.toml`. You must pass `-m "$MODEL"`, and the value must be a model accepted by the authenticated Codex account. List options with:

   ```bash
   codex debug models | jq -r '.models[].slug'
   ```

   `gpt-5.6-sol` is a known capable starting point if your account offers it.

1. **Create a Worktree.** The orchestrator (you) creates and removes the worktree; the worker only receives the path.

   ```bash
   git worktree add -b fanout-test ./.worktrees/wt-a main
   ```

1. **Dispatch Your First Worker.** Put the brief in a file, then set absolute paths and run the command:

   ```bash
   export WORKTREE="$(pwd)/.worktrees/wt-a"
   export BRIEF="$(pwd)/.worktrees/brief-a.md"
   export LOG="$(pwd)/.worktrees/worker-a.jsonl"
   export ERR="$(pwd)/.worktrees/worker-a.stderr"
   export REPORT="$(pwd)/.worktrees/worker-a.report"
   export MODEL="gpt-5.6-sol"

   timeout 1500 codex exec \
     -C "$WORKTREE" \
     -m "$MODEL" \
     --ignore-user-config \
     --ignore-rules \
     --ephemeral \
     --sandbox workspace-write \
     --json \
     --output-last-message "$REPORT" \
     - < "$BRIEF" \
     > "$LOG" 2> "$ERR"
   STATUS=$?
   echo "exit=$STATUS"
   ```

1. **Verify.** The cost or token metadata in the JSONL is the CLI's own view and may not match your invoice, so never quote it as fact. Read the report, then run the tests yourself, because a green run from a worker proves the worker's tests agree with the worker's code and nothing else:

   ```bash
   test -s "$REPORT"
   jq -e -c . < "$LOG" > /dev/null
   jq -e -s 'all(.[]; ((.type // "") != "error" and (.type // "") != "turn.failed"))' "$LOG" > /dev/null
   ! grep -iE -q 'rejected a tool call|sandbox.*denied|permission.*denied|approval.*required|network.*blocked' "$ERR"
   git -C "$WORKTREE" status --porcelain
   ```

## Which Model to Use

`$MODEL` is required. `--ignore-user-config` also discards the default model in your `config.toml`, so the worker cannot fall back to it. The value you pass to `-m` must be a model id accepted by the local Codex account. Use `codex debug models | jq -r '.models[].slug'` to see the available choices, and pick one before dispatching.

| Model id      | Notes                                                  |
| ------------- | ------------------------------------------------------ |
| `gpt-5.6-sol` | A capable starting point used in this repo's smoke test |
| any slug from `codex debug models` | Must be accepted by your authenticated account         |

The cheapest or fastest model for your task depends on the current Codex model catalog; re-check before relying on one.

## How It Stays Safe

A worker runs under one of the sandbox modes below. For normal work, use `--sandbox workspace-write`; the `--dangerously-bypass-approvals-and-sandbox` flag is only for a disposable scratch directory.

| Mode / flag                                  | Auto-approves                   | Use When                                                                  |
| -------------------------------------------- | ------------------------------- | ------------------------------------------------------------------------- |
| `workspace-write`                            | Reads and writes in the cwd     | **The default.** Files only; every Bash command is sandboxed              |
| `--dangerously-bypass-approvals-and-sandbox` | Everything                      | Only in a scratch directory or worktree you will throw away               |
| `read-only`                                  | Nothing; every prompt is denied | Read-only analysis. Anything not allowed is refused, and it carries on    |

- **Give a Worker a `git worktree`, Never Your Checkout.** A half-done or looping run then costs one worktree removal, not a reconstruction, and two workers never share a working tree.
- **Say `do not run git` in Every Brief.** The worker can, and a commit from a worker is a commit nobody reviewed.
- **Do not add `--ask-for-approval` to `codex exec`.** That flag is for interactive `codex`; it is invalid for `codex exec` and will be rejected.

## What It Cannot Do

- **The Cost Metadata Is Not an Invoice.** The JSONL reports cost and token figures from the CLI's point of view and they may not match your actual usage bill. Never quote them as fact.
- **The Full Config Is Expensive.** A worker without `--ignore-user-config --ignore-rules` can load every plugin, rule, and MCP server you have, increasing token use and wall time. Always run workers with the flags shown above.
- **Six Concurrent Workers Is the Ceiling.** Past that they contend for the same files and the review cost exceeds the saving.
- **A Stuck Worker Still Spends Quota.** The `timeout 1500` wrapper is a wall-clock guard, not a turn budget. Exit 124 means the guard fired.
- **A Worker Has None of Your Context.** Everything it needs goes in the brief, and if the brief takes longer to write than the task takes to do, do the task.

## Under the Hood

Every worker is one command, and every failure shows up around it:

<details>
<summary><b>The Dispatch Command</b></summary>

```bash
timeout 1500 codex exec \
  -C "$WORKTREE" \
  -m "$MODEL" \
  --ignore-user-config \
  --ignore-rules \
  --ephemeral \
  --sandbox workspace-write \
  --json \
  --output-last-message "$REPORT" \
  - < "$BRIEF" \
  > "$LOG" 2> "$ERR"
STATUS=$?
echo "exit=$STATUS"
```

- **`-` Reads the Brief From stdin.** Put the brief in a file; an inline prompt of any length is shell-quoting archaeology.
- **`-C` Sets the Working Directory First.** The worker's world is its cwd: that is where it reads `AGENTS.md` and where relative paths in the brief resolve.
- **Always Redirect to a Log File.** `codex exec --json` emits JSONL events, one per line, and `--output-last-message` writes the final response to `$REPORT`.
- **Always Wrap in `timeout`.** A stuck or looping worker can still spend model quota until 1500 seconds. Exit 124 means the timeout fired.
- **`$MODEL` Is Required.** `--ignore-user-config` also discards the default model in `config.toml`, so the orchestrator must choose one.

</details>

<details>
<summary><b>Failure Modes</b></summary>

| Symptom                                                  | Cause And Fix                                                                                    |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Exit 124                                                 | The 1500-second wall-clock timeout fired. Inspect `$LOG` and `$ERR` before retrying.             |
| Non-zero exit, or missing/empty `$REPORT`                | The run failed or was killed. Do not merge the worktree.                                         |
| `rejected a tool call`, `sandbox.*denied`, or `permission.*denied` on stderr | `--sandbox workspace-write` refused a command. Use `--add-dir` or do that step yourself.         |
| Unexpected files or a commit in the repo                 | The brief did not say "and nothing else" or "do not run git". `git status` after every run.      |
| Every turn costs many tokens                             | The worker did not run with `--ignore-user-config --ignore-rules`. Check the flags.              |
| Model id rejected by the CLI                             | The `-m` value is not accepted by the local Codex account. Re-check `codex debug models`.         |
| `network.*blocked` on stderr                             | The worker's environment or sandbox blocked a network call the brief required.                   |

</details>

## Go Deeper

| Read                   | When                                                                                     |
| ---------------------- | ---------------------------------------------------------------------------------------- |
| [`SKILL.md`](SKILL.md) | You are writing a brief or a report: preflight, dispatch, verification and failure modes |

## Contributing

Issues and pull requests are welcome.

## Licence

[MIT](LICENSE). Copyright 2026 TolongLabs.
