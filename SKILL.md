---
name: codex-fanout
description: "Use when a task splits into independent chunks that need a capable model but not your judgement, so the coordinator can fan the work out to headless `codex exec` workers in isolated Git worktrees and review the results."
---

# Codex Fan-Out

Codex CLI is the coordinator. The fan-out child workers are headless `codex exec` processes. Each child works in its own isolated Git worktree, writes a JSONL log and a final report, and waits for the coordinator to review.

```
Codex CLI coordinator
        ↓ loads this skill, writes briefs, starts workers
`codex exec` workers
        ↓ -C "$WORKTREE" --sandbox workspace-write
isolated Git worktrees
        ↓ JSONL log + report
coordinator review
```

![Codex fanout architecture](assets/codex-fanout-diagram.png)

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

**1. Check that Codex CLI is installed.**

```bash
codex --version
```

**2. Check authentication with `codex login status`.** If the account is not authenticated, tell the coordinator to run `codex login`.

```bash
codex login status
```

**3. Choose a model.** List available slugs from the current Codex catalog. If the CLI help shows a different current command, use that instead.

```bash
codex debug models | jq -r '.models[].slug'
```

Use a model already named by the user as the standing choice. If the user has not named one, ask one concise normal-conversation question before dispatching and offer the available model slugs. Do not start a worker without a selected model.

---

## Dispatch

### The Command

Use absolute paths for every variable. Each worker gets its own `$WORKTREE`, `$BRIEF`, `$REPORT`, `$LOG`, and `$ERR`.

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

- **`-C "$WORKTREE"`** selects the worker cwd. It must be an isolated Git worktree, given as an absolute path.
- **`-m "$MODEL"`** is explicit because `--ignore-user-config` ignores the coordinator's default model; the worker must know which model to use.
- **`--ignore-user-config`** and **`--ignore-rules`** make the worker run with the repository rules that are in the brief only.
- **`--ephemeral`** avoids persisting session files to disk.
- **`--sandbox workspace-write`** is the normal worker sandbox: it can write files inside the worktree but not escape it.
- **`--json`** emits JSONL on stdout.
- **`--output-last-message "$REPORT"`** writes the worker's final message to a report file.
- **`- < "$BRIEF"`** reads the brief from a file on stdin. Put the brief in a file; inline prompts are shell-quoting archaeology.
- **`> "$LOG" 2> "$ERR"`** captures stdout and stderr separately. Always redirect; the JSONL result can be long.
- **`timeout`** protects against a worker that hangs on a long call or retry loop. Exit 124 means the timeout fired.

### Permission And Sandbox Choices

| Flag / mode                                | What it allows                            | Use when                                                                  |
| ------------------------------------------ | ----------------------------------------- | ------------------------------------------------------------------------- |
| `--sandbox read-only`                      | Reads the workspace; no writes            | Analysis, summaries, inventories                                          |
| `--sandbox workspace-write`                | Edits inside the workspace                | **The default.** Files in the worktree only; shell commands stay sandboxed |
| `--dangerously-bypass-approvals-and-sandbox` | Runs commands without approval or sandbox | Only in a scratch directory or worktree you will throw away               |

- **Read-only mode is for analysis.** Use `--sandbox read-only` when the worker only needs to read and report.
- **`--dangerously-bypass-approvals-and-sandbox` is for disposable scratch work only.** It removes all guards; never point it at a real repository.
- **Never add interactive approval flags to a headless command.** Flags like `--approve-for-me` require an interactive terminal.
- **Never give two workers overlapping files.** Each worktree and brief are separate.

### Git Worktrees

Give a worker a Git worktree, never the coordinator's checkout. A half-done or looping run then costs one worktree removal, not a reconstruction.

```bash
git worktree add -b <branch> <scratch>/wt-<chunk> <base-branch>
```

Then point the worker at the absolute path of that worktree with `-C`.

### Parallel, Up To Six

Start at most six background copies of the command. Each gets its own brief file, log, report, and worktree. **Never let two workers write the same file.**

Wait for all processes to finish before reviewing; do not poll.

### Sequential

Chain in one shell command when later chunks depend on earlier output, or when workers touch overlapping files. One set of logs per stage, `&&` between them, so a failure stops the chain.

**Prefer parallel.** Sequential is for real dependencies, not for tidiness.

---

## Writing The Brief

A worker prompt is a work order. Six things, and the first two are what actually prevent damage:

1. **Name every file to create or edit, and say "and nothing else".** Without it you get stray scratch files.
2. **State inputs as paths inside the working directory, and state the output format concretely.** If the output is committed, say "read them, never mention their paths in your output" so machine paths do not leak into the deliverable.
3. **Give the output format concretely.** Heading levels, table columns, casing. "Well structured" produces whatever the model likes today.
4. **Carry repository-specific rules the worktree does not already state.** The repository's own rule files are already in the worktree; repeat only what is specific to this job.
5. **Say what must be preserved verbatim** when the task is a transformation. Models summarise by reflex.
6. **Ask for a short report** - what it wrote, what it could not do, what it guessed at. It arrives as the message written by `--output-last-message`.

Say **"do not run git"** in every brief. The worker can, and a commit from a worker is a commit nobody reviewed.

---

## Verifying, Which Is Not Optional

Do not present worker cost or token metadata from the JSONL log as an invoice. Those numbers are diagnostic, not a bill.

Check, in this order:

```bash
test -f "$REPORT"                                           # the worker produced a report
jq -R 'fromjson?' "$LOG" | grep -E '"type":"error"' || true   # parse JSONL and detect error events
grep -iE "denied|blocked|refused" "$ERR" || true            # the sandbox refused a command
git -C "$WORKTREE" status --porcelain                       # what actually changed, including strays
<run the relevant test command in "$WORKTREE">              # verify behaviour independently
grep -RIn -E '/home/|C:\\Users|/tmp/' "$WORKTREE"        # no machine paths leaked
```

Then **read the parts that carry risk**, run the tests yourself, and mutation-test any test the worker wrote. A green run from a worker proves the worker's tests agree with the worker's code and nothing else.

Fix small defects yourself. Re-dispatch only if a chunk is broadly wrong, with the defect named in the new brief.

---

## Failure Modes Seen In The Wild

| Symptom                                                  | Cause and fix                                                                                    |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Exit 124 and nothing written                             | The timeout fired. The worker hung or retried. Read `$ERR` and `$LOG` for the cause              |
| `denied`, `blocked`, or `refused` in `$ERR`              | The sandbox refused a command. Either run the command yourself or adjust the brief               |
| Report file missing                                      | The worker exited before the final message. Check `$LOG` and `$ERR` for the failure event        |
| JSONL contains error events                              | A tool call or model error happened during the run. Read the events in `$LOG`                    |
| `git status` shows unexpected files or a commit          | The brief did not say "and nothing else" or "do not run git". Check after every run              |
| `permission_denials` or sandbox refusals in the JSONL    | `workspace-write` refused a command outside the worktree. Keep the worker inside its worktree    |
| Model slug rejected by Codex                             | Catalogue drift. Re-list `codex debug models` rather than retrying the same slug                 |
| Stray machine paths in the deliverable                   | The brief did not say "read them, never mention their paths in your output". Filter before review |
| Worker wrote outside its worktree                        | The worktree path was not absolute or the wrong sandbox was used. Use `-C` with an absolute path  |

---

## Reporting Back

Say which model ran, how many workers, what each produced, **and what you corrected**. The corrections are the useful part - they tell the user whether the next fan-out should use a stronger model or a tighter brief.

Never present a worker's output as verified when you only checked that the file exists.
