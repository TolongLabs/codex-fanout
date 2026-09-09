---
name: codex-fanout
description: "Fan work out to headless Claude Code workers running a cheap OpenRouter model through CLIProxyAPI, up to 6 at once. This skill is loaded and invoked by the Codex CLI coordinator; the workers are Claude Code processes behind the proxy. Use when a task splits into independent chunks that need a capable model but not your judgement. Requires the proxy to be up and a model chosen before dispatching."
---

# Codex Fan-Out

Codex CLI is the harness that loads and invokes this skill. The fan-out workers are still headless Claude Code runs, pointed at CLIProxyAPI so they spend a few cents of OpenRouter credit instead of the Claude plan.

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

The worker command is `claude -p`. Do not substitute `codex exec`: the workers are Claude Code processes behind the proxy, not Codex CLI children.

Each worker is the full Claude Code agent in one directory; you write the brief, it writes the files, you review.

**Why headless Claude Code.** The `claude -p` harness gives you `--max-turns`, a real exit code, JSON output, and it reads `AGENTS.md` and `CLAUDE.md` on its own, so briefs carry the task and not the house rules.

**Six concurrent workers is the ceiling.** Past that they contend for the same files and the review cost exceeds the saving.

---

## When This Pays, And When It Does Not

| Delegate                                                          | Keep                                                    |
| ----------------------------------------------------------------- | ------------------------------------------------------- |
| Converting a dump into structured Markdown                        | Deciding what the structure should be                   |
| The same transformation across many files                         | Anything where being wrong is expensive and quiet       |
| Drafting from a spec you have already written                     | Writing the spec                                        |
| Per-file summaries, inventories, mechanical extraction            | Architecture, naming, API shape, security               |
| A code change whose exact diff and tests are already in the brief | Work needing conversation context the worker cannot see |

**A worker has none of your context.** Everything it needs goes in the brief. If the brief takes longer to write than the task takes to do, do the task.

---

## Preflight, In Order

**1. Bring the proxy up and get its token.** No shell alias is involved; these are the raw pieces.

```bash
CLIPROXY_DIR="${CLIPROXY_DIR:-$HOME/.cli-proxy-api}"
CLIPROXY_PORT="${CLIPROXY_PORT:-8317}"
CLIPROXY_TOKEN=$(grep -A1 '^api-keys:' "$CLIPROXY_DIR/config.yaml" | tail -1 | tr -d '" -')
(exec 3<>"/dev/tcp/127.0.0.1/$CLIPROXY_PORT") 2>/dev/null || {
  nohup "${CLIPROXY_BIN:-$HOME/.local/opt/cliproxyapi/cli-proxy-api}" -config "$CLIPROXY_DIR/config.yaml" \
    >> "$CLIPROXY_DIR/proxy.log" 2>&1 & sleep 3
}
```

Never print the token. It is a local secret and it leaks into transcripts.

**2. List the OpenRouter models the proxy serves.** Only the `openrouter` block of `config.yaml` counts. The proxy also serves whatever OAuth logins it holds, and those are not worker models: they are not cheap, and a dead login fails every call.

```bash
sed -n '/name: "openrouter"/,/^  - name: "/p' "$CLIPROXY_DIR/config.yaml" | grep -E '^\s+alias:' | tr -d '" ' | cut -d: -f2
```

**3. Ask the user which model to use, unless they already named one.** In Codex, ask one concise question in the normal conversation, with the default first, offering only aliases listed in step 2. A model the user named earlier in the same conversation is a standing answer: state it back and dispatch.

- **`glm-5.3-flash`** - the default. Fast, cheap, and the one every measurement in this file was taken on
- Anything else in the list, by its alias

You must choose a model before dispatching.

**Adding a cheap worker model.** Every model below supports tool calls and a large context, and costs under a dollar per million output tokens. Prices are OpenRouter's on 2026-09-07; re-check before relying on one.

| OpenRouter id                       | Input / output per 1M tokens | Context | Notes                                    |
| ----------------------------------- | ---------------------------- | ------- | ---------------------------------------- |
| `z-ai/glm-5.3-flash`                | $0.075 / $0.25               | 1.3M    | The default; measured in this file       |
| `qwen/qwen3.7-flash`                | $0.03 / $0.13                | 1M      | Cheapest capable option                  |
| `deepseek/deepseek-v4-flash`        | $0.08 / $0.16                | 1M      | Cheap output, long context               |
| `qwen/qwen3-coder-30b-a3b-instruct` | $0.07 / $0.28                | 262k    | Coder-tuned                              |
| `google/gemini-2.5-flash-lite`      | $0.10 / $0.40                | 1M      | Fast                                     |
| `minimax/minimax-m3`                | $0.30 / $1.20                | 1M      | The step-up when flash models fall short |

To add one, append it under the `openrouter` block's `models:` list in `config.yaml` and restart the proxy:

```yaml
      - name: "qwen/qwen3.7-flash"
        alias: "qwen3.7-flash"
        display-name: "Qwen3.7 Flash"
```

```bash
kill "$(cat "$CLIPROXY_DIR/proxy.pid")" 2>/dev/null; pkill -x cli-proxy-api; sleep 1   # then run step 1 again
```

**4. Give workers their own config dir, once per machine.** A worker under your normal `~/.claude` loads every plugin and hook you have: measured 105,726 input tokens per turn against 19,027 with an empty config dir, and twice the wall time. DKM and claude-mem also fire inside the worker, which you do not want.

```bash
export CLAUDE_FANOUT_CONFIG="$HOME/.claude-fanout"
mkdir -p "$CLAUDE_FANOUT_CONFIG"
```

An empty directory is a complete config: no plugins, no hooks, no memory. The worker still reads the repository's `CLAUDE.md` and `AGENTS.md` from the working directory.

---

## Dispatch

### The Command

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

- **`-p` reads the brief from stdin.** Put the brief in a file; an inline prompt of any length is shell-quoting archaeology
- **`cd` into the working directory first.** The worker's world is its cwd: that is where it reads `AGENTS.md` and `CLAUDE.md`, and where relative paths in the brief resolve
- **Always redirect to a log file.** The JSON result is the last line, several kilobytes long; stderr warnings come before it. Read it with `tail -n 1`, never `tail -c`
- **Always wrap in `timeout`.** On a proxy error Claude Code retries for about three minutes before giving up, and a looping worker burns `--max-turns` worth of credit. Exit 124 means the timeout fired
- **`--max-turns` is the budget.** Forty is enough for a multi-file edit with tests; ten for a single-file rewrite
- **`$MODEL` is required.** The default is `glm-5.3-flash`, but you must choose a model before dispatching, and use a model the user explicitly named when one already exists in the conversation

### Choosing `--permission-mode`

| Mode                | Auto-approves                   | Use When                                                                  |
| ------------------- | ------------------------------- | ------------------------------------------------------------------------- |
| `acceptEdits`       | Reads and edits in the cwd      | **The default.** Files only; every Bash command is refused unless allowed |
| `bypassPermissions` | Everything                      | Only in a scratch directory or worktree you will throw away               |
| `default`           | Nothing; every prompt is denied | Read-only analysis. Anything not allowed is refused, and it carries on    |

`acceptEdits` refuses every Bash command, measured: a worker's three `awk` line-width checks were all denied. Add `--allowedTools "Bash(bun test:*)"` for the commands a brief asks the worker to run, and nothing wider.

**Give a worker a `git worktree`, never your checkout.** A half-done or looping run then costs one worktree removal, not a reconstruction, and two workers never share a working tree:

```bash
git worktree add -b <branch> <scratch>/wt-<chunk> main
```

### Parallel, Up To Six

Start at most six background copies of the `claude -p` command. Each gets its own brief file, its own log, its own worktree or output paths. **Never let two workers write the same file.**

Wait for all processes to finish before reviewing; do not poll.

### Sequential

Chain in one shell command when later chunks depend on earlier output, or when workers touch overlapping files. One log per stage, `&&` between them, so a failure stops the chain.

**Prefer parallel.** Sequential is for real dependencies, not for tidiness.

---

## Writing The Brief

A worker prompt is a work order. Six things, and the first two are what actually prevent damage:

1. **Name every file to create or edit, and say "and nothing else".** Without it you get stray scratch files
2. **State inputs as paths inside the working directory**, and if the output is committed, say "read them, never mention their paths in your output" - otherwise machine paths leak into the deliverable
3. **Give the output format concretely.** Heading levels, table columns, casing. "Well structured" produces whatever the model likes today
4. **Carry in the house rules the repository does not already state.** The worker reads `AGENTS.md` and `CLAUDE.md` itself; repeat only what is specific to this job
5. **Say what must be preserved verbatim** when the task is a transformation. Models summarise by reflex
6. **Ask for a short report** - what it wrote, what it could not do, what it guessed at. It arrives as the `result` field of the JSON line at the end of the log

Say **"do not run git"** in every brief. The worker can, and a commit from a worker is a commit nobody reviewed.

---

## Verifying, Which Is Not Optional

The JSON result reports `total_cost_usd` as if Anthropic served the model. **It is not the upstream OpenRouter invoice.** Real cost is on the proxy's upstream. Never quote it.

Check, in this order:

```bash
tail -n 1 "$LOG" | jq -r '.result, .num_turns, .permission_denials'      # its report, turns, refusals
git -C <worktree> status --porcelain                                   # what actually changed, including strays
grep -c "<structural marker>" <output>                                 # right shape, right count
grep -rn "/home/\|C:\\\\Users\|/tmp/" <output>                         # no machine paths leaked
```

Then **read the parts that carry risk**, run the tests yourself, and mutation-test any test the worker wrote. A green run from a worker proves the worker's tests agree with the worker's code and nothing else.

Fix small defects yourself. Re-dispatch only if a chunk is broadly wrong, with the defect named in the new brief.

---

## Failure Modes Seen In The Wild

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

---

## Reporting Back

Say which model ran, how many workers, what each produced, **and what you corrected**. The corrections are the useful part - they tell the user whether the next fan-out should use a stronger model or a tighter brief.

Never present a worker's output as verified when you only checked that the file exists.
