---
name: codex-delegate
description: Use when a task is execution-shaped rather than decision-shaped — running experiments, iterating on a script until it runs, building visualizations or HTML viewers, configuring environments, bulk reruns, log spelunking — and it would consume many turns in the main session. Covers when to delegate, how to write an execution contract, how to run long jobs in the background, and how to collect results without polluting context. Also use when the main session's context has grown past roughly 150K and cost is being driven by context re-reads.
---

# Delegating execution to Codex

Claude's expensive resource is not its output — it is **turn count × context size**. A
200K-context session costs ~200K tokens *per assistant turn*, whether the turn writes one
line or one thousand. The fix is not "write less"; it is **do fewer turns in a big
context**.

This skill moves turn-heavy execution into Codex, which runs with a fresh context and its
own budget.

## Invoke Codex directly — do not use `codex-companion.mjs`

Call the CLI:

```bash
codex exec [OPTIONS] <PROMPT>
```

**Do not route through `codex-companion.mjs`** (the `/codex:rescue` path). It hardcodes
`sandbox: request.write ? "workspace-write" : "read-only"`. Codex's `workspace-write`
sandbox is implemented with bubblewrap, which fails outright in containers where
unprivileged user namespaces are blocked:

```
bwrap: No permissions to create a new namespace
```

`/root/.codex/config.toml` already sets `sandbox_mode = "danger-full-access"` and
`approval_policy = "never"`. Calling `codex exec` directly means Codex honours that config
and never invokes bwrap. Verified working.

`codex-companion.mjs` has no flag to override the sandbox, and patching it is pointless —
plugin updates overwrite it.

## Decision table

| Task | Who |
|---|---|
| Running an experiment / training / inference | **Codex** |
| Writing a script, then running it, reading the error, editing, rerunning (≥2 iterations) | **Codex** |
| Building an HTML/JS visualization or viewer | **Codex** |
| Environment setup, dependency resolution, conda env work | **Codex** |
| Bulk reruns, sweeps, multi-seed/multi-config launches | **Codex** |
| Reading logs to find why a run failed | **Codex** |
| Long refactors across many files | **Codex** |
| Deciding *what* to run and *why* | **Claude** |
| Interpreting results, judging whether a difference matters | **Claude** |
| Priorities, tradeoffs, planning, writing the research narrative | **Claude** |
| Talking with the user | **Claude** |

The dividing line: **Claude owns the decision and the verdict; Codex owns the doing.**

If a task is mostly Claude reading its own context to decide the next command, it is
execution-shaped. Delegate it.

## Writing an execution contract

Codex starts with a fresh context. It knows none of this session's reasoning. Everything
it needs must be in the contract — and nothing else. A good contract is short, concrete,
and checkable.

```markdown
# Goal
One sentence. What must be true when this is done.

# Where
- Code root: /abs/path
- Inputs: /abs/path (which files, which config)
- Outputs go to: /abs/path/outputs/<name>/   (create it; do not write elsewhere)

# Environment
- conda env: <name>   (verify with `conda env list` before assuming)
- GPUs: use <which>; do not run DL code on CPU
- Launch long runs with nohup + a log file, never blocking

# Steps
1. ...
2. ...

# Design and scope
- Required behavior and design decisions: <what must be implemented and why>
- Allowed code changes and constraints: <files/components; behavior to preserve>

# Success criteria
- <objective, checkable> — e.g. "4 result dirs exist, each with metrics.json"

# Code change handoff (required whenever code changes, including repair runs)
- Before editing, record the baseline commit and working-tree status; save the pre-edit diff or contents of files you will touch, including existing untracked files, under the output directory. Preserve unrelated user work. Without Git, save before-copies of files to be edited there.
- In your final reply (saved as CODEX_REPLY.md), list every added/modified/deleted source, configuration, or documentation file, relevant symbols or line ranges, what changed and why, and which design requirement each change satisfies.
- Identify the baseline and resulting commits/working-tree changes, including new files, so Claude can locate the full diff even if you committed your changes.
- Report commands/checks actually run and their results, checks not run, deviations, risks, and unresolved work. Explicitly state if no code changed.
- Your completion report is a handoff for Claude's review, not acceptance. Every repair requires an updated report; do not claim Claude has approved it.

# Failure handling
- On error: fix and retry up to N times.
- If still failing: STOP, write the full error to <path>/BLOCKED.md, report. Do not improvise a workaround.
- Review fixes share the contract's retry limit. If blocked or a design/scope change is needed, report it to Claude; do not silently change acceptance criteria.

# Rules
- Do not edit files outside the paths above.
- Do not delete raw data.
- Do not add fallbacks.
- Report: what ran, what the numbers are, what failed.
```

Two rules that matter more than they look:

- **Name the failure path.** Without it, Codex invents workarounds and you get silent
  garbage instead of a clear stop.
- **Make success objective.** "Figure out why it's slow" is not checkable; "4 result dirs
  with metrics.json" is.

Write it to a file. Passing a long prompt as a shell argument invites escaping bugs, and
the file is itself a reusable artifact:

```bash
codex exec -C /abs/path --skip-git-repo-check - < /abs/path/contract.md
```

## Three recipes

### 1. Short, bounded task — run it in the foreground

```bash
codex exec -C /abs/path --skip-git-repo-check \
  --output-last-message /abs/path/outputs/<name>/CODEX_REPLY.md \
  - < /abs/path/contract.md
```

`--output-last-message` writes Codex's final answer to a file. Read that file rather than
letting a long transcript land in your context. That is the single cheapest token
optimization in this skill.

### 2. Long job (GPU runs, sweeps) — background it, arm a watcher, leave

```bash
cd /abs/path && nohup codex exec -C /abs/path --skip-git-repo-check \
  --output-last-message /abs/path/outputs/<name>/CODEX_REPLY.md \
  - < /abs/path/contract.md > /abs/path/outputs/<name>/codex.log 2>&1 &
CODEX_PID=$!
```

Then immediately arm a watcher that exits on **any** terminal state — not just success:

```bash
until [ -s /abs/path/outputs/<name>/CODEX_REPLY.md ] \
   || [ -s /abs/path/outputs/<name>/BLOCKED.md ] \
   || ! kill -0 $CODEX_PID 2>/dev/null; do sleep 30; done
```

Run that watcher with `Bash` and `run_in_background: true`. It re-invokes you the moment
it exits, which is the completion notification. Write the watcher so the reason for exit is
visible:

```bash
R=/abs/path/outputs/<name>; until [ -s "$R/CODEX_REPLY.md" ] || [ -s "$R/BLOCKED.md" ] || ! kill -0 $CODEX_PID 2>/dev/null; do sleep 30; done; \
  if [ -s "$R/CODEX_REPLY.md" ]; then echo "codex DONE -> $R/CODEX_REPLY.md"; \
  elif [ -s "$R/BLOCKED.md" ]; then echo "codex BLOCKED -> $R/BLOCKED.md (see codex.log)"; \
  else echo "codex DIED without output -> $R/codex.log"; fi
```

Three branches, deliberately. `--output-last-message` only fires on a clean finish, so
watching only for that file means a crashed job looks identical to a finished one.

Record the PID. Then *stop working on that task in this session*. Do not poll in a loop —
each poll is a full turn at full context price, which is the exact cost this skill exists
to avoid. Move to another phase, or tell the user what is running and end the turn.

To collect:

```bash
tail -40 /abs/path/outputs/<name>/codex.log     # progress and errors
cat /abs/path/outputs/<name>/CODEX_REPLY.md     # Codex's own summary
```

Read those two files. Do not paste the whole log into context.

**The watcher is best-effort.** It dies with the session, and `Bash` background tasks carry
a timeout — re-arm it if a job outlives it. The on-disk `CODEX_REPLY.md` /
`BLOCKED.md` are the durable fallback: **at the start of any session, check for reply files
from jobs you launched earlier before assuming anything is still running.**

Use `Monitor`, not a backgrounded `Bash`, only when you want *periodic progress events*
while a job runs (it caps at 30 minutes). For a single completion signal, backgrounded
`Bash` is the right tool.

### 3. Follow-up fix on the same work — resume, don't restart

```bash
cd /abs/path && codex exec resume '<SESSION_ID>' \
  --output-last-message /abs/path/outputs/<name>/CODEX_REPLY.md \
  - < /abs/path/outputs/<name>/review-fixes.md
```

Record the session ID from the original run; replace `<SESSION_ID>` with that exact ID.
Write the findings, expected behavior, and updated-handoff requirement to `review-fixes.md`.
Do not select an unrelated session with `--last` when several tasks have run.

Resume keeps Codex's accumulated context on the codebase. Use it for "rerun with 1024",
"now render the multi-view visualization", "apply the top fix". Restarting from scratch
throws away everything Codex already learned.

Unrelated new work: start a fresh `codex exec` (do not pass `resume`).

## Required code review before acceptance

Every code-changing Codex delivery, including a follow-up fix, must pass Claude's review.
Put the handoff requirements above into each execution contract and carry them into repair
instructions. A successful exit, a completion notification, or `CODEX_REPLY.md` alone does
not establish that the code implements the requested design.

1. **Collect the handoff.** Read `CODEX_REPLY.md`, any `BLOCKED.md`, and the relevant
   validation results. If the change report or baseline is missing, obtain it before
   accepting the work. Code changes delivered with a failure or partial result still
   need review; report their status honestly.
2. **Read the actual code.** Inspect the full task diff against the recorded baseline and
   enough surrounding code to understand behavior. Include committed, staged, unstaged,
   and new/untracked files; a clean working tree or plain `git diff` can omit changes.
   Separate Codex's edits from pre-existing user work. Without Git, compare the saved
   before-copies and resulting files. If the actual changes cannot be established, mark
   the review blocked instead of approving from the summary.
3. **Compare with the contract.** Check each design requirement, allowed scope, preserved
   behavior, and acceptance criterion against the implementation and evidence. Look for
   missing work, unintended changes, and relevant correctness or regression risks. Run
   only focused checks needed to resolve uncertainty; do not automatically repeat costly
   experiments or create a broad test suite for a small change. Disclose unverified items.
4. **Record Claude's verdict.** Save a concise `CLAUDE_REVIEW.md` in the task output
   directory: reviewed revision/diff, `PASS`, `NEEDS_CHANGES`, or `BLOCKED`, requirement
   coverage, validation evidence/limits, and any findings with file/line and reason.
   `PASS` requires the agreed design and acceptance criteria to be satisfied; Codex must
   not author Claude's approval. Any later code change requires another review.
5. **Repair and re-review.** Send concrete findings and expected behavior back to the
   specific Codex session that produced the work. Require an updated handoff, inspect the
   repair and its effects on the full task, and update the verdict. Respect the contract's
   retry limit. If the limit is reached or a decision is needed, report the blocker or ask
   for the missing decision; do not relax the design or add a fallback to obtain a pass.

Only after `PASS` may Claude tell the user the code-change task is complete. Until then,
report it as awaiting review, needing changes, or blocked. The user-facing result should
briefly state what changed, Claude's review conclusion, and any validation limits.

## Useful flags

| Flag | Why |
|---|---|
| `-C, --cd <DIR>` | working root — always set it explicitly |
| `--skip-git-repo-check` | needed when the work dir is not a git repo |
| `-o, --output-last-message <FILE>` | final answer to a file; keeps context clean |
| `--json` | JSONL events, when you need to parse results |
| `--output-schema <FILE>` | force a JSON-schema-shaped final answer |
| `--add-dir <DIR>` | extra writable directories |
| `-c key=value` | override any config.toml value for this run |
| `-m, --model <MODEL>` | e.g. `gpt-6-astra` (the configured default) |

Do **not** pass `-s danger-full-access` — config.toml already sets it, and being explicit
invites questions about whether the default was deliberate.

## Context discipline — the other half of the cost

Delegation cuts turns. Session boundaries cut context size. You need both.

- **Start a new session per phase.** Reading a report, surveying literature, running an
  experiment, and building a viewer are four unrelated topics; carrying them in one
  context means every turn pays for all four.
- **Reconcile outstanding jobs first.** Any watcher armed in a previous session is dead. On
  entering a session, look for `CODEX_REPLY.md` and `BLOCKED.md` under the output dirs of
  jobs you launched earlier, and check whether those PIDs are still alive. Do not assume a
  job is still running, and do not assume it finished.
- **Before switching, write the conclusions down.** Put them in `MEMORY.md` or a findings
  file so the next session starts informed instead of re-deriving.
- **Signal to the user.** At the end of a phase, say what is done and suggest `/clear`.
  Do not silently carry on.
- **Rough threshold:** past ~150K context, or past ~40 assistant turns on one topic, the
  next phase belongs in a new session.

## Anti-patterns

- **Delegating the decision.** "Which of these four failure modes should I chase?" is
  Claude's job. Claude can investigate one you already chose.
- **Delegating a one-liner.** `git status`, `ls`, reading one file — just do it. The
  contract overhead exceeds the work.
- **Polling in the main session.** Every poll is a full turn at full context price.
- **Watching only for the success file.** `--output-last-message` is written only on a
  clean finish. A watcher that waits for `CODEX_REPLY.md` alone stays silent when a job
  dies, and silence then reads as success. Watch `BLOCKED.md` and process death too.
- **Using `Monitor` for a one-shot completion signal.** It caps at 30 minutes. A
  backgrounded `Bash` until-loop that exits on the condition is the tool built for this.
- **Pasting the whole log back.** Read the tail and `CODEX_REPLY.md`.
- **Omitting the failure path.** Codex will improvise; you get a plausible wrong answer
  instead of an honest stop.
- **Resuming the wrong task.** Use the recorded session ID for repairs; a changed topic
  needs a fresh session.
- **Delegating then re-implementing the same work.** Review the actual code and return
  concrete fixes to Codex instead of duplicating its implementation. Required review is
  part of Claude's job and must not be skipped to save turns.
- **Routing through `codex-companion.mjs` / `/codex:rescue`.** See above — it forces the
  bwrap sandbox and fails.

## Reporting

The user's global rules require telling them after any command runs, including those Codex
ran. When reporting a delegated task, state: the PID, what Codex was told to do, where
output lands, and — once collected — what the numbers actually were. Do not paraphrase a
result you have not read.
