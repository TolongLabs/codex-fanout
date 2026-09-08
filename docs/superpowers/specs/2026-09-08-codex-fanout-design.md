# Codex Fan-Out Design

**Date:** 2026-09-08

**Source model:** Adapt `https://github.com/TolongLabs/claude-fanout` at source revision
`f1431e2f6ba0fd0fc9130981dde06ff4b476a644` for the OpenAI Codex CLI harness.

## Goal

Create a standalone `codex-fanout` skill repository with the same public shape and operating model as
`claude-fanout`: brief-driven delegation, isolated worker directories, bounded parallelism, captured worker output,
and independent verification. Every provider-specific instruction must target Codex CLI only.

## Scope

The published repository will retain the source repository's five-file shape:

- `README.md` — standalone installation, quick start, operating model, safety notes, failure modes, and license links.
- `SKILL.md` — the concise Codex skill loaded by an agent when fan-out is appropriate.
- `LICENSE` — MIT license with the project copyright.
- `.gitignore` — ignore local worker/test artifacts.
- `assets/codex-fanout-banner.png` — a Codex-branded adaptation of the source hero banner.

The source repository's Claude Code, CLIProxyAPI, Anthropic, Claude config, and Claude permission terminology will not
appear in the resulting operational instructions. The process-only design record under `docs/superpowers/specs/` is
committed while this work is planned, then removed from the final public tree so the published surface remains
source-compatible.

The implementation must use this source-to-Codex mapping:

| Source concept | Codex-only replacement |
| --- | --- |
| Claude Code `claude -p` | `codex exec` |
| CLIProxyAPI/OpenRouter model catalog | the locally configured Codex model, optionally selected with `-m` |
| `CLAUDE_CONFIG_DIR` | `--ignore-user-config` while retaining `CODEX_HOME` authentication |
| Claude `acceptEdits` | `--sandbox workspace-write` |
| Claude `--max-turns` | external `timeout 1500` |
| Claude final JSON result | Codex JSONL from `--json` plus the final response from `--output-last-message` |

The published `README.md` and `SKILL.md` must not contain operational references to `Claude`, `Claude Code`,
`CLIProxyAPI`, `Anthropic`, `OpenRouter`, `ANTHROPIC_*`, `CLAUDE_CONFIG_DIR`, `claude -p`, `acceptEdits`, or
`--max-turns`. The source URL may appear only in the design record, which is removed before publication.

## Codex-specific operating model

Workers run with `codex exec`, not Claude Code. The documented default invocation will:

- run from a dedicated Git worktree using `-C`;
- isolate user configuration with `--ignore-user-config` while retaining the user's Codex authentication;
- avoid persistent worker sessions with `--ephemeral`;
- use `--sandbox workspace-write` for normal worktree edits; `codex exec` is non-interactive, so the command must not
  include the interactive-only `--ask-for-approval` flag;
- emit machine-readable JSONL with `--json`;
- save the final worker response with `--output-last-message`;
- wrap the process in `timeout 1500` because Codex CLI has no Claude-style `--max-turns` flag;
- accept the brief through stdin from a file and redirect stdout/stderr to a per-worker log.

`CODEX_HOME` remains the source of authentication. `--ignore-user-config` is the documented isolation mechanism for
not loading the operator's normal `config.toml`, MCP servers, and other user-level configuration into workers.

The normative command template is:

```bash
timeout 1500 codex exec \
  -C "$WORKTREE" \
  --ignore-user-config \
  --ephemeral \
  --sandbox workspace-write \
  --json \
  --output-last-message "$REPORT" \
  - < "$BRIEF" \
  > "$LOG" 2>&1
STATUS=$?
```

`$BRIEF`, `$LOG`, and `$REPORT` are absolute paths; `$WORKTREE` is the worker's dedicated Git worktree. The optional
`-m "$MODEL"` is placed after `codex exec` when the operator selects a model. The command assumes Codex CLI 0.153.4
or newer and requires implementers to re-check `codex exec --help` if the CLI changes.

The skill will document `--dangerously-bypass-approvals-and-sandbox` only for disposable scratch directories, never as
the normal repository mode. It will explain that worker model selection is an optional `-m` argument governed by the
installed Codex CLI, not by a proxy model catalog.

## Isolation and concurrency

The worker contract will mirror the source repository:

- maximum six concurrent workers;
- one brief, log, report, and worktree per worker;
- no overlapping file ownership between workers;
- every brief names the exact files to touch, says “and nothing else,” and says “do not run git”;
- the orchestrating agent reviews the worker output and performs commits/merges.

The orchestrating agent creates each worktree before dispatch with
`git worktree add -b <branch> <scratch>/wt-<worker> <base-ref>` and removes it after review with
`git worktree remove <scratch>/wt-<worker>`. The worker only receives the resulting worktree path; it never creates,
merges, or removes worktrees. The README and skill will show the single-worker command and parallelization rules. They
will not implement a long-running scheduler or daemon.

Sequential dispatch for dependent chunks will remain supported by running the same command one worker at a time and
using the prior worktree or merged result as the next worker's base. Parallel dispatch is preferred only when file
ownership is independent.

For parallel dispatch, the orchestrator creates `wt-a` through `wt-f`, gives each worker a distinct brief/log/report
triple, starts at most six copies of the normative command as background jobs, waits for all process statuses, then
reviews each worktree independently. No worker may write another worker's worktree or any shared report path.

The README and skill will document the same failure contract: a timeout is exit 124; any non-zero worker exit is
unverified; a missing or empty final-response file is a failed worker; rejected tool calls or sandbox denials must be
reported; unexpected files, commits, or edits outside the brief are review failures; and the orchestrator must inspect
`git status` and run tests independently before merging.

On any of those failures, the orchestrator does not merge the worktree. It preserves the log and report for diagnosis,
then either removes the disposable worktree after inspection or re-runs with a corrected brief; retries are explicit,
not automatic. A worker that ran `git` or created unexpected files is treated as untrusted until the orchestrator has
reviewed and cleaned the worktree.

## Verification

The external smoke test will run a real `codex exec` worker in `/tmp/codex-fanout-smoke-<pid>`, without adding a
scheduler or test framework to the published five-file tree. Its fixture is exact:

- `AGENTS.md`: “For this smoke test, create only `worker-output.txt` with exactly `codex-fanout smoke pass` followed
  by a newline. Do not edit any other file and do not run git.”
- tracked `sentinel.txt`: exactly `untouched` followed by a newline;
- `brief.md`: “Read `AGENTS.md`. Create exactly the file it requests. Do not run git. Report the created filename and
  its exact contents.”

The test initializes the fixture on branch `main`, creates worktree
`/tmp/codex-fanout-smoke-<pid>/wt-smoke` with
`git worktree add -b smoke-worker .../wt-smoke main`, then runs the documented command with
`--ignore-user-config`, `--ephemeral`, `--sandbox workspace-write`, `--json`, `--output-last-message`, and
`timeout 1500`; it passes
only when exit status is zero, exit status is not 124, the final-response file is non-empty, no error/denial event is
present in the JSONL log, the expected file has exact contents, the sentinel is unchanged, and `git status` shows only
the requested file. A second read-only check will run the published skill validator:
`python /home/adam/.codex/skills/.system/skill-creator/scripts/quick_validate.py /home/adam/.codex/skills/codex-fanout`
and require exit status zero. The skill is installed for that check by copying the published tree to
`/home/adam/.codex/skills/codex-fanout` after the process-only spec file has been removed.

`SKILL.md` is the normative runtime contract; `README.md` is the standalone public explanation and must keep its
command, flags, safety model, and failure semantics aligned with `SKILL.md`. The banner is non-executable presentation
content and will be created by editing the source banner's composition or generating an equivalent image; it will be
accepted if it is a readable horizontal Codex Fan-Out image with no Claude/provider branding. `LICENSE` will name
TolongLabs as copyright holder, and the final push assumes GitHub credentials are already available to Git.

The published `.gitignore` will contain concrete patterns for `reference/`, `.worktrees/`, `*.log`, `*.jsonl`, and
`*.report`, keeping local fan-out artifacts out of the source-compatible tree.

## Delivery

After implementation and verification:

1. install/copy the skill into the local Codex skills directory for a real harness invocation;
2. run the smoke test from `/home/adam/CS/sandbox/codex-fanout` and review all generated artifacts;
3. remove the process-only design record from the published tree, while retaining its earlier commit;
4. commit the repository on `main`;
5. push `main` to `https://github.com/TolongLabs/codex-fanout.git`.
