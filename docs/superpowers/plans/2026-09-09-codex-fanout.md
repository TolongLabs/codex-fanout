# Codex Fan-Out Implementation Plan

> **For agentic workers:** REQUIRED: Use the approved `codex-fanout` design and the `devin-fanout` workflow for the delegated implementation. Do not add a scheduler or commit worker changes from Devin.

**Goal:** Publish a standalone `codex-fanout` skill repository that mirrors the TolongLabs source repository while using only the Codex CLI harness.

**Architecture:** `SKILL.md` is the normative runtime contract and `README.md` is its standalone public mirror. Workers are launched with `codex exec` from isolated Git worktrees, with per-worker briefs, JSONL logs, stderr logs, and final-response reports. The published tree contains only the source-compatible docs, license, ignore file, and Codex banner; smoke-test artifacts stay outside the repository.

**Tech Stack:** Markdown skill documentation, Bash command templates, Git worktrees, Codex CLI 0.153.4+, Python skill validator, PNG banner.

---

## Chunk 1: Published skill and repository packaging

### Task 1: Prepare the source reference

**Files:**
- Create ignored reference tree: `reference/claude-fanout/`
- Read: `reference/claude-fanout/README.md`, `reference/claude-fanout/SKILL.md`, `reference/claude-fanout/LICENSE`, `reference/claude-fanout/.gitignore`

- [ ] **Step 1: Clone the pinned source**

Run from the repository root:

```bash
git clone https://github.com/TolongLabs/claude-fanout.git reference/claude-fanout
git -C reference/claude-fanout checkout f1431e2f6ba0fd0fc9130981dde06ff4b476a644
```

Expected: the reference checkout is at the pinned commit and is ignored by the repository.

- [ ] **Step 2: Compare the source public shape**

Run:

```bash
find reference/claude-fanout -maxdepth 2 -type f -not -path '*/.git/*' -print | sort
```

Expected: the source has `README.md`, `SKILL.md`, `LICENSE`, `.gitignore`, and `assets/claude-fanout-banner.png`.

### Task 2: Write the Codex skill contract

**Files:**
- Create: `SKILL.md`

- [ ] **Step 1: Write the frontmatter and trigger boundary**

Use exactly `name: codex-fanout` and a trigger-focused description beginning with `Use when`; state that this applies to independent chunks suitable for headless Codex CLI workers and does not replace coordinator judgement.

- [ ] **Step 2: Port the source operating model**

Retain the source sections for delegation boundaries, preflight, dispatch, permission/sandbox choices, parallel and sequential operation, brief writing, verification, failure modes, and reporting. Replace every provider-specific mechanism with the normative Codex command from the design:

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
```

Explain that `$MODEL` is required, `$CODEX_HOME` retains authentication, `--ignore-user-config` and `--ignore-rules` isolate worker configuration while `AGENTS.md` still loads from the worktree, `timeout 1500` is a wall-clock guard, and `--json` is JSONL rather than one final JSON object.

- [ ] **Step 3: Add the exact safety and verification contract**

Document: six-worker ceiling; one worktree/brief/log/error/report per worker; `git worktree add` and `git worktree remove` are orchestrator responsibilities; no overlapping ownership; “and nothing else” plus “do not run git” in every brief; no merge on non-zero/124 exit, missing report, denial, unexpected files, or failed tests; preserve logs for diagnosis; independently run tests; and never quote worker cost metadata.

Also document `--dangerously-bypass-approvals-and-sandbox` only for disposable scratch directories, never for a normal repository worktree.

- [ ] **Step 4: Check the forbidden-term boundary**

Run a case-insensitive scan over `SKILL.md` and ensure operational content contains none of: `Claude`, `Claude Code`, `CLIProxyAPI`, `Anthropic`, `OpenRouter`, `ANTHROPIC_`, `CLAUDE_CONFIG_DIR`, `claude -p`, `acceptEdits`, or `--max-turns`.

### Task 3: Write the standalone README and metadata

**Files:**
- Create: `README.md`
- Create: `.gitignore`
- Create: `LICENSE`

- [ ] **Step 1: Mirror the source README structure**

Keep the source's standalone shape and sections: hero image, badges, summary, contents, what it does, quick start, model choice, safety, limits, under the hood, deeper reading, contributing, and license. Use Codex-only badges and wording. The quick-start installation path must be `${CODEX_HOME:-$HOME/.codex}/skills/codex-fanout`.

- [ ] **Step 2: Keep README and SKILL command text aligned**

Use the same `codex exec` flags, environment semantics, worktree lifecycle, failure contract, and JSONL verification rules in both files. The README may explain more, but it must not contradict `SKILL.md`.

- [ ] **Step 3: Add concrete ignore and license files**

Use `.gitignore` patterns for `reference/`, `.worktrees/`, `*.log`, `*.jsonl`, and `*.report`. Copy the MIT license text with `Copyright (c) 2026 TolongLabs`.

### Task 4: Add the Codex hero asset

**Files:**
- Create: `assets/codex-fanout-banner.png`

- [ ] **Step 1: Create a 1600×620 horizontal banner**

Adapt the source composition or generate an equivalent dark navy terminal/worktree fan-out visual. Center the exact readable title `CODEX FAN-OUT`; do not include Claude/provider branding.

- [ ] **Step 2: Inspect the asset**

Confirm the PNG is readable at normal README width and has the expected dimensions.

### Task 5: Validate the published tree before smoke testing

**Files:**
- Test: published tree contents and documentation scans

- [ ] **Step 1: Run the skill validator**

Run:

```bash
python /home/adam/.codex/skills/.system/skill-creator/scripts/quick_validate.py .
```

Expected: exit 0 with no frontmatter or scaffold-placeholder errors.

- [ ] **Step 2: Run documentation integrity checks**

Check `git diff --check`, compare command blocks in `README.md` and `SKILL.md`, scan forbidden terms, and verify that only the intended published files plus the process spec/plan are present before final cleanup.

## Chunk 2: Real Codex harness smoke test and delivery

### Task 6: Remove process-only files and install the skill

**Files:**
- Delete from published tree: `docs/superpowers/specs/2026-09-08-codex-fanout-design.md`
- Delete from published tree: `docs/superpowers/plans/2026-09-09-codex-fanout.md`
- Remove ignored working reference after review: `reference/claude-fanout/`

- [ ] **Step 1: Preserve the planning commits and clean the public tree**

Remove the process-only docs from the working tree without rewriting earlier commits. Move any ignored reference checkout out of the repository so the final tree contains only the five source-compatible files and asset.

- [ ] **Step 2: Install the exact published tree**

Copy the final skill directory to `/home/adam/.codex/skills/codex-fanout` for the real Codex harness test. Do not copy the removed process files.

### Task 7: Run the real Codex smoke test

**Files:**
- External temporary script: `/tmp/codex-fanout-smoke.sh`
- External fixture: `/tmp/codex-fanout-smoke-<pid>/`

- [ ] **Step 1: Write the reproducible smoke script**

Create `/tmp/codex-fanout-smoke.sh` as an executable script with `set -euo pipefail`, a disposable root at
`/tmp/codex-fanout-smoke-$$`, and cleanup only after assertions pass. The script must contain the fixture creation,
worktree creation, normative dispatch, assertions, and cleanup from the following steps; it is not added to the
published repository.

- [ ] **Step 2: Create the exact fixture**

Run `git init -b main`; create `AGENTS.md` with exactly: `For this smoke test, create only worker-output.txt with exactly codex-fanout smoke pass followed by a newline. Do not edit any other file and do not run git.` Create tracked `sentinel.txt` containing exactly `untouched\n`; create `brief.md` requesting that exact task; commit the fixture.

- [ ] **Step 3: Create the disposable worktree**

Run:

```bash
git worktree add -b smoke-worker /tmp/codex-fanout-smoke-<pid>/wt-smoke main
```

- [ ] **Step 4: Run the documented command**

Use `MODEL="${CODEX_FANOUT_MODEL:-gpt-5.6-sol}"`, absolute `$BRIEF`, `$LOG`, `$ERR`, and `$REPORT` paths, and the exact normative command. Capture the process status and preserve the artifacts until all checks pass.

- [ ] **Step 5: Assert observable behavior**

Run `test "$STATUS" -eq 0`; `test -s "$REPORT"`; `jq -e -c . < "$LOG" > /dev/null`; `jq -e -s 'all(.[]; ((.type // "") != "error" and (.type // "") != "turn.failed"))' "$LOG" > /dev/null`; `! grep -iE -q 'rejected a tool call|sandbox.*denied|permission.*denied|approval.*required|network.*blocked' "$ERR"`; `diff -u <(printf 'codex-fanout smoke pass\n') "$WORKTREE/worker-output.txt"`; `diff -u <(printf 'untouched\n') "$WORKTREE/sentinel.txt"`; and `test "$(git -C "$WORKTREE" status --porcelain)" = "?? worker-output.txt"`.

- [ ] **Step 6: Remove smoke artifacts**

After recording the evidence, remove the disposable worktree, fixture directory, and `/tmp/codex-fanout-smoke.sh`. Preserve no generated logs in the published tree.

### Task 8: Review, commit, and push

**Files:**
- Final published tree: `README.md`, `SKILL.md`, `.gitignore`, `LICENSE`, `assets/codex-fanout-banner.png`

- [ ] **Step 1: Review the final diff and status**

Run `git status --short`, `git diff --check`, `git diff --stat`, forbidden-term scans, and the validator against `/home/adam/.codex/skills/codex-fanout`.

- [ ] **Step 2: Commit the published skill**

Create a Conventional Commit such as, after confirming `git status --short` contains only the intended published files and the two planned deletions:

```bash
git add -A
git commit -m "feat: add codex fanout skill"
```

- [ ] **Step 3: Verify the remote before pushing**

Run `git remote get-url origin` and confirm the output is `https://github.com/TolongLabs/codex-fanout.git`.

- [ ] **Step 4: Push main**

Run `git push -u origin main`, then verify the remote ref with:

```bash
test "$(git ls-remote origin refs/heads/main | awk '{print $1}')" = "$(git rev-parse main)"
```

Report the pushed commit SHA and the smoke-test evidence; do not claim success from the push exit code alone without the ref check.
