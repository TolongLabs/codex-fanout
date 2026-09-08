# Codex Fan-Out Design

**Date:** 2026-09-08

**Source model:** Adapt the TolongLabs `claude-fanout` standalone skill repository for the OpenAI Codex CLI harness.

## Goal

Create a standalone `codex-fanout` skill repository with the same public shape and operating model as
`claude-fanout`: brief-driven delegation, isolated worker directories, bounded parallelism, captured worker output,
and independent verification. Every provider-specific instruction must target Codex CLI only.

## Scope

The repository will retain the source repository's five-file shape:

- `README.md` — standalone installation, quick start, operating model, safety notes, failure modes, and license links.
- `SKILL.md` — the concise Codex skill loaded by an agent when fan-out is appropriate.
- `LICENSE` — MIT license with the project copyright.
- `.gitignore` — ignore local worker/test artifacts.
- `assets/codex-fanout-banner.png` — a Codex-branded adaptation of the source hero banner.

The source repository's Claude Code, CLIProxyAPI, Anthropic, Claude config, and Claude permission terminology will not
appear in the resulting operational instructions. The repository may also contain the design record under
`docs/superpowers/specs/` because this workflow requires the approved design to be committed before implementation.

## Codex-specific operating model

Workers run with `codex exec`, not Claude Code. The documented default invocation will:

- run from a dedicated Git worktree using `-C`;
- isolate user configuration with `--ignore-user-config` while retaining the user's Codex authentication;
- avoid persistent worker sessions with `--ephemeral`;
- use `--sandbox workspace-write` and `--ask-for-approval never` for normal worktree edits;
- emit machine-readable JSONL with `--json`;
- save the final worker response with `--output-last-message`;
- wrap the process in `timeout` because Codex CLI has no Claude-style `--max-turns` flag;
- accept the brief through stdin from a file and redirect stdout/stderr to a per-worker log.

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

The README and skill will show both a single-worker command and the parallelization rules. They will not implement a
long-running scheduler or daemon.

## Verification

The repository will include a small smoke-test path that runs a real `codex exec` worker in a disposable Git worktree.
Verification will inspect the process exit code, the final-response file, JSONL/log errors, worktree status, and the
expected worker artifact. The test will prove the actual Codex command shape rather than merely matching documentation
text. The completed skill will also pass the Codex skill validator.

## Delivery

After implementation and verification:

1. install/copy the skill into the local Codex skills directory for a real harness invocation;
2. run the smoke test in `/home/adam/CS/sandbox/codex-fanout` and review all generated artifacts;
3. commit the repository on `main`;
4. push `main` to `https://github.com/TolongLabs/codex-fanout.git`.

