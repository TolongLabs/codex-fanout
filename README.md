![Codex Fan-Out](assets/codex-fanout-banner.png)

# Codex Fan-Out

![Codex CLI skill](https://img.shields.io/badge/Codex_CLI_skill-000000?style=for-the-badge)
![CLIProxyAPI](https://img.shields.io/badge/CLIProxyAPI-000000?style=for-the-badge)
![MIT licence](https://img.shields.io/badge/MIT_licence-blue?style=for-the-badge)

**Fan work out to headless Claude Code workers running a cheap OpenRouter model through CLIProxyAPI, up to six at once.**

> The worker does the bulk, you keep the judgement.

_Fan-out_ is dispatching many independent workers at once and reviewing what comes back. The Codex CLI coordinator loads this skill and starts the workers; the workers are still Claude Code processes behind the proxy.

```
Codex CLI coordinator
        ↓ loads this skill, writes briefs, starts workers
shell / env dispatch
        ↓ env -u ANTHROPIC_API_KEY ... timeout 1500 claude -p ...
headless Claude Code (claude -p)
        ↓ ANTHROPIC_BASE_URL + ANTHROPIC_AUTH_TOKEN
local CLIProxyAPI (port 8317)
        ↓ upstream request
OpenRouter model
```

The worker command is `claude -p`. Do not substitute `codex exec`.

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

- **Codex CLI coordinator.** This skill is installed in the Codex CLI skills directory and is invoked as `$codex-fanout`. Codex CLI is the harness that loads the skill, writes the brief, and dispatches the workers.
- **Headless Claude Code Workers on a Cheap Proxy Model.** Each worker is the full Claude Code agent in one directory, pointed at CLIProxyAPI so it spends a few cents of OpenRouter credit instead of the Claude plan.
- **You Write the Brief, It Writes the Files, You Review.** The worker reads the repository's `AGENTS.md` and `CLAUDE.md` on its own, so the brief carries the task and not the house rules.
- **Up to Six at Once.** Launch them in parallel, each with its own brief, log and worktree, and never let two workers write the same file.
- **A Real Harness.** `claude -p` gives you `--max-turns`, a real exit code and JSON output, and it reads `AGENTS.md` and `CLAUDE.md` on its own.

If a task needs your judgement or your conversation context, you do not need this skill.

## Quick Start

1. **Check the Prerequisites.** You need Codex CLI, Claude Code, CLIProxyAPI installed and configured with at least one OpenRouter model, and `curl` and `jq`.

1. **Clone Into the Global Skills Directory**, then restart or reload Codex CLI so the skill loads. It is available as `$codex-fanout` from then on.

   ```bash
   git clone https://github.com/TolongLabs/codex-fanout "${CODEX_HOME:-$HOME/.codex}/skills/codex-fanout"
   ```

1. **Bring the Proxy Up and Choose a Model.** Start the proxy if its port is closed, then list the OpenRouter models it serves. Only the `openrouter` block of its config counts. Never print the token, which is a local secret and leaks into transcripts.

   ```bash
   CLIPROXY_DIR="${CLIPROXY_DIR:-$HOME/.cli-proxy-api}"
   CLIPROXY_PORT="${CLIPROXY_PORT:-8317}"
   CLIPROXY_TOKEN=$(grep -A1 '^api-keys:' "$CLIPROXY_DIR/config.yaml" | tail -1 | tr -d '" -')
   (exec 3<>"/dev/tcp/127.0.0.1/$CLIPROXY_PORT") 2>/dev/null || {
     nohup "${CLIPROXY_BIN:-$HOME/.local/opt/cliproxyapi/cli-proxy-api}" -config "$CLIPROXY_DIR/config.yaml" \
       >> "$CLIPROXY_DIR/proxy.log" 2>&1 & sleep 3
   }
   ```

   ```bash
   sed -n '/name: "openrouter"/,/^  - name: "/p' "$CLIPROXY_DIR/config.yaml" | grep -E '^\s+alias:' | tr -d '" ' | cut -d: -f2
   ```

   `glm-5.3-flash` is the default. You must pick one before dispatching, and use a model the user already named in the conversation if one exists.

1. **Export the Empty Worker Config, once per machine.** A worker under your normal `~/.claude` loads every plugin, hook and memory you have; an empty config dir avoids that. The worker still reads the repository's `AGENTS.md` and `CLAUDE.md`.

   ```bash
   export CLAUDE_FANOUT_CONFIG="$HOME/.claude-fanout"
   mkdir -p "$CLAUDE_FANOUT_CONFIG"
   ```

1. **Give the Worker a Git Worktree, Never Your Checkout.** Then `cd` into it, put the brief in a file, pick a model from the previous step, and dispatch:

   ```bash
   git worktree add -b fanout-test ./.worktrees/wt-a main
   ```

   ```bash
   cd ./.worktrees/wt-a

   env -u ANTHROPIC_API_KEY \
     CLAUDE_CONFIG_DIR="$CLAUDE_FANOUT_CONFIG" \
     ANTHROPIC_BASE_URL="http://127.0.0.1:$CLIPROXY_PORT" \
     ANTHROPIC_AUTH_TOKEN="$CLIPROXY_TOKEN" \
     timeout 1500 claude -p \
       --model "$MODEL" \
       --permission-mode acceptEdits \
       --max-turns 40 \
       --output-format json \
       < "brief.md" \
       > "worker.log" 2>&1
   echo "exit=$?"
   ```

   `$MODEL` must be set before you run this. If the user already named a model in the conversation, use it; otherwise `glm-5.3-flash` is the default.

1. **Verify.** The `total_cost_usd` in the JSON result is not the upstream OpenRouter invoice, so never quote it. Read the report, then run the tests yourself, because a green run from a worker proves the worker's tests agree with the worker's code and nothing else:

   ```bash
   tail -n 1 "worker.log" | jq -r '.result, .num_turns, .permission_denials'
   git -C ./.worktrees/wt-a status --porcelain
   grep -c "<structural marker>" <output>
   grep -rn "/home/\|C:\\\\Users\|/tmp/" <output>
   ```

## Which Model to Use

Only models in the `openrouter` block of the proxy config are offered, and `glm-5.3-flash` is the default. Any of these is cheap enough to be a worker; prices are OpenRouter's on 2026-09-07, so re-check before relying on one. `SKILL.md` shows the three lines that add one to the proxy.

| OpenRouter id                       | Input / output per 1M tokens | Context | Notes                                    |
| ----------------------------------- | ---------------------------- | ------- | ---------------------------------------- |
| `z-ai/glm-5.3-flash`                | $0.075 / $0.25               | 1.3M    | The default; measured in this file       |
| `qwen/qwen3.7-flash`                | $0.03 / $0.13                | 1M      | Cheapest capable option                  |
| `deepseek/deepseek-v4-flash`        | $0.08 / $0.16                | 1M      | Cheap output, long context               |
| `qwen/qwen3-coder-30b-a3b-instruct` | $0.07 / $0.28                | 262k    | Coder-tuned                              |
| `google/gemini-2.5-flash-lite`      | $0.10 / $0.40                | 1M      | Fast                                     |
| `minimax/minimax-m3`                | $0.30 / $1.20                | 1M      | The step-up when flash models fall short |

## How It Stays Safe

A worker writes files under one of three permission modes:

| Mode                | Auto-Approves                   | Use When                                                                  |
| ------------------- | ------------------------------- | ------------------------------------------------------------------------- |
| `acceptEdits`       | Reads and edits in the cwd      | **The default.** Files only; every Bash command is refused unless allowed |
| `bypassPermissions` | Everything                      | Only in a scratch directory or worktree you will throw away               |
| `default`           | Nothing; every prompt is denied | Read-only analysis. Anything not allowed is refused, and it carries on    |

- **Give a Worker a `git worktree`, Never Your Checkout.** A half-done or looping run then costs one worktree removal, not a reconstruction, and two workers never share a working tree.
- **Say `do not run git` in Every Brief.** The worker can, and a commit from a worker is a commit nobody reviewed.
- **Add `--allowedTools` Narrowly if a Command Is Required.** `acceptEdits` refuses Bash by default. Add something like `--allowedTools "Bash(bun test:*)"` for the one command the brief needs, and nothing wider.

## What It Cannot Do

- **The Cost Figure Is Not the Upstream Invoice.** The JSON result reports `total_cost_usd` as if Anthropic served the model; the real cost is on the proxy's upstream. Never quote it.
- **The System Prompt Is Not Free.** A worker under your normal config dir loads every plugin and hook you have. In the source repo this measured 105,726 input tokens per turn against 19,027 with an empty one, and twice the wall time.
- **Six Concurrent Workers Is the Ceiling.** Past that they contend for the same files and the review cost exceeds the saving.
- **A Proxy Error Costs About Three Minutes.** On one, Claude Code retries until it gives up, and a looping worker burns `--max-turns` worth of credit, which is why every dispatch wraps in `timeout`.
- **A Worker Has None of Your Context.** Everything it needs goes in the brief, and if the brief takes longer to write than the task takes to do, do the task.

## Under the Hood

Every worker is one command, and every failure shows up around it:

<details>
<summary><b>The Dispatch Command</b></summary>

```bash
env -u ANTHROPIC_API_KEY \
  CLAUDE_CONFIG_DIR="$CLAUDE_FANOUT_CONFIG" \
  ANTHROPIC_BASE_URL="http://127.0.0.1:$CLIPROXY_PORT" \
  ANTHROPIC_AUTH_TOKEN="$CLIPROXY_TOKEN" \
  timeout 1500 claude -p \
    --model "$MODEL" \
    --permission-mode acceptEdits \
    --max-turns 40 \
    --output-format json \
    < "<path to the brief>" \
    > "<log path>" 2>&1
echo "exit=$?"
```

- **`-p` Reads the Brief From stdin.** Put the brief in a file; an inline prompt of any length is shell-quoting archaeology.
- **`cd` Into the Working Directory First.** The worker's world is its cwd: that is where it reads `AGENTS.md` and `CLAUDE.md`, and where relative paths in the brief resolve.
- **Always Redirect to a Log File.** The JSON result is the last line, several kilobytes long, and stderr warnings come before it. Read it with `tail -n 1`, never `tail -c`.
- **Always Wrap in `timeout`.** On a proxy error Claude Code retries for about three minutes before giving up, and a looping worker burns `--max-turns` worth of credit. Exit 124 means the timeout fired.
- **`--max-turns` Is the Budget.** Forty is enough for a multi-file edit with tests; ten for a single-file rewrite.
- **`$MODEL` Is Required.** `glm-5.3-flash` is the default, but you must choose one before dispatching and use a model the user explicitly named in the conversation if one exists.

</details>

<details>
<summary><b>Failure Modes</b></summary>

| Symptom                                                  | Cause And Fix                                                                                    |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `auth_unavailable: no auth available (providers=...)`    | You named a model outside the `openrouter` block, and its login is dead. Use an OpenRouter alias |
| Exit 124, `terminal_reason":"api_error`, nothing written | The proxy rejected every call and the harness retried until the timeout. Fix the proxy first     |
| `[claude-code:unrecognized_model]` on stderr             | Harmless. Claude Code does not know the proxy model's name; the call still goes through          |
| `claude.ai connectors are disabled` on stderr            | Harmless. `ANTHROPIC_AUTH_TOKEN` takes precedence over the login, which is the point             |
| Every turn costs ~100k input tokens                      | The worker ran under your normal config dir. Set `CLAUDE_CONFIG_DIR` to the empty one            |
| Stray files or a commit in the repo                      | The brief did not say "and nothing else" or "do not run git". `git status` after every run       |
| `permission_denials` is non-empty                        | `acceptEdits` refused a command. Either allow it with `--allowedTools` or do that step yourself  |
| Model id rejected by the proxy                           | Catalogue drift. Re-list `/v1/models` rather than retrying the same id                           |

</details>

## Go Deeper

| Read                   | When                                                                                     |
| ---------------------- | ---------------------------------------------------------------------------------------- |
| [`SKILL.md`](SKILL.md) | You are writing a brief or a report: preflight, dispatch, verification and failure modes |

## Contributing

Issues and pull requests are welcome.

## Licence

[MIT](LICENSE). Copyright 2026 TolongLabs.
