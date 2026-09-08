---
name: codex-fanout
description: "Use when a task splits into independent chunks that need a capable model but not your judgement, so you can fan the work out to headless Codex CLI workers running from isolated Git worktrees, up to 6 at once."
---

# Codex Fan-Out

Dispatch headless `codex exec` runs as background workers from isolated Git worktrees. Each worker is the Codex CLI agent in one directory; you write the brief, it writes the files, you review.

**Why `codex exec`.** The harness gives you a real exit code, JSONL output, and it reads `AGENTS.md` on its own, so briefs carry the task and not the house rules.

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

**1. Check the Codex CLI and authentication.** The worker needs a local Codex install and a logged-in account. `CODEX_HOME` (default `~/.codex`) holds authentication and state; do not share credentials in briefs or logs.

```bash
codex --version
codex login status
```

If `codex login status` is non-zero, run `codex login`.

**2. Choose a model.** Because the worker runs with `--ignore-user-config --ignore-rules`, it does not load your default model from `config.toml`. You must pass `-m "$MODEL"` and the value must be a model accepted by the authenticated Codex account. List options with:

```bash
codex debug models | jq -r '.models[].slug'
```

Pick one before dispatching. `gpt-5.6-sol` is a known capable starting point if your account offers it.

**3. Create the worker's Git worktree.** The orchestrator (you) creates and removes worktrees; the worker only receives the path.

```bash
git worktree add -b <branch> <scratch>/wt-<chunk> <base-ref>
```

Use an absolute path for the worktree. Create the directories that will hold `$LOG`, `$ERR`, and `$REPORT` as well.

---

## Dispatch

### The Command

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
```

- `$BRIEF`, `$LOG`, `$ERR`, and `$REPORT` are absolute paths.
- `$WORKTREE` is the worker's dedicated Git worktree.
- `$MODEL` is required because user config is ignored.
- The command assumes Codex CLI 0.153.4 or newer. Re-check `codex exec --help` if the CLI changes.
- Do not add `--ask-for-approval` to `codex exec`; it is interactive-only and invalid here.

### What the flags do

- **`-C` sets the working directory** to the worktree. The worker's world is its cwd: that is where it reads `AGENTS.md` and where relative paths in the brief resolve.
- **`-m` sets the model explicitly.** It is required because `--ignore-user-config` also discards your default model.
- **`--ignore-user-config` and `--ignore-rules`** keep your `config.toml`, MCP servers, user rules, and project execpolicy `.rules` files out of the worker; `AGENTS.md` in the worktree still loads.
- **`--ephemeral`** avoids persistent worker sessions.
- **`--sandbox workspace-write`** allows file edits in the workspace and is the normal mode.
- **`--json`** emits JSONL events, one per line.
- **`--output-last-message`** writes the final response to `$REPORT`.

### Choosing `--sandbox`

| Mode / flag                                         | Auto-approves                   | Use When                                                                  |
| --------------------------------------------------- | ------------------------------- | ------------------------------------------------------------------------- |
| `workspace-write`                                   | Reads and writes in the cwd     | **The default.** Files only; every Bash command is sandboxed              |
| `--dangerously-bypass-approvals-and-sandbox`        | Everything                      | Only in a scratch directory or worktree you will throw away               |
| `read-only`                                         | Nothing; every prompt is denied | Read-only analysis. Anything not allowed is refused, and it carries on    |

`workspace-write` can still refuse a Bash command that tries to leave the workspace or touch disallowed paths. Either rephrase the brief, use `--add-dir` for an explicit extra directory, or do that step yourself.

### Parallel, Up To Six

Start at most six background copies of the command. Each gets its own brief file, its own log, its own error file, its own report, and its own worktree. **Never let two workers write the same file.**

Wait for all processes to finish before reviewing; do not poll.

### Sequential

Chain one worker at a time when later chunks depend on earlier output, or when workers touch overlapping files. One log per stage, and a failure stops the chain.

**Prefer parallel.** Sequential is for real dependencies, not for tidiness.

---

## Writing The Brief

A worker prompt is a work order. Six things, and the first two are what actually prevent damage:

1. **Name every file to create or edit, and say "and nothing else".** Without it you get stray scratch files.
2. **State inputs as paths inside the working directory**, and if the output is committed, say "read them, never mention their paths in your output" — otherwise machine paths leak into the deliverable.
3. **Give the output format concretely.** Heading levels, table columns, casing. "Well structured" produces whatever the model likes today.
4. **Carry in the house rules the repository does not already state.** The worker reads `AGENTS.md` itself; repeat only what is specific to this job.
5. **Say what must be preserved verbatim** when the task is a transformation. Models summarise by reflex.
6. **Ask for a short report** — what it wrote, what it could not do, what it guessed at. It arrives in `$REPORT`.

Say **"do not run git"** in every brief. The worker can, and a commit from a worker is a commit nobody reviewed.

---

## Verifying, Which Is Not Optional

The cost or token metadata in the JSONL is the CLI's own view and may not match your invoice. **Do not quote it as fact.**

Check, in this order:

```bash
test -s "$REPORT"                                                     # final response exists and is non-empty
jq -e -c . < "$LOG" > /dev/null                                       # every stdout line is valid JSON
jq -e -s 'all(.[]; ((.type // "") != "error" and (.type // "") != "turn.failed"))' "$LOG" > /dev/null
! grep -iE -q 'rejected a tool call|sandbox.*denied|permission.*denied|approval.*required|network.*blocked' "$ERR"
git -C "$WORKTREE" status --porcelain                                 # what actually changed, including strays
```

Then **read the parts that carry risk**, run the tests yourself, and mutation-test any test the worker wrote. A green run from a worker proves the worker's tests agree with the worker's code and nothing else.

Fix small defects yourself. Re-dispatch only if a chunk is broadly wrong, with the defect named in the new brief.

---

## Failure Modes Seen In The Wild

| Symptom                                                   | Cause And Fix                                                                                      |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Exit 124                                                  | The 1500-second wall-clock timeout fired. A stuck worker can still spend model quota until then.   |
| Non-zero exit, or missing/empty `$REPORT`                 | The run failed or was killed. Do not merge; inspect `$LOG` and `$ERR`.                             |
| `rejected a tool call`, `sandbox.*denied`, or `permission.*denied` in `$ERR` | `--sandbox workspace-write` refused a command. Rephrase, use `--add-dir`, or do that step yourself. |
| Unexpected files or a commit in the worktree              | The brief did not say "and nothing else" or "do not run git". `git status` after every run.        |
| Every turn costs many tokens                              | The worker did not run with `--ignore-user-config --ignore-rules`. Check the flags.                |
| Model id rejected by the CLI                              | The value passed to `-m` is not accepted by the local Codex account. Re-check `codex debug models`. |
| `network.*blocked` in `$ERR`                              | The worker's environment or sandbox blocked a network call the brief required.                     |

---

## Reporting Back

Say which model ran, how many workers, what each produced, **and what you corrected**. The corrections are the useful part — they tell the user whether the next fan-out should use a stronger model or a tighter brief.

Never present a worker's output as verified when you only checked that the file exists.

---

## Git Worktree Lifecycle

The orchestrator creates the worktree before dispatch and removes it after review:

```bash
# before
git worktree add -b <branch> <scratch>/wt-<chunk> <base-ref>

# after successful review
git worktree remove <scratch>/wt-<chunk>
```

The worker never creates, merges, or removes worktrees. Do not merge a worktree whose verification failed. On failure, preserve `$LOG`, `$ERR`, and `$REPORT` for diagnosis, then remove the disposable worktree or re-run with a corrected brief; retries are explicit, not automatic.
