---
name: codex-delegate
description: Use when a task is execution-shaped rather than decision-shaped — running experiments, iterating on a script until it runs, building visualizations or HTML viewers, configuring environments, bulk reruns, log spelunking — and it would consume many turns in the main session. Covers when to delegate, how to write an execution contract, how to run long jobs in the background, and how to collect results. Also use when the main session's context has grown past roughly 150K and cost is being driven by context re-reads.
---

# Delegating execution to Codex

Claude's expensive resource is not its output — it is **turn count × context size**. A
200K-context session costs ~200K tokens *per assistant turn*, whether the turn writes one
line or one thousand. So the rule is not "write less"; it is **do fewer turns in a big
context**.

This skill moves turn-heavy execution out of the main session and into Claude, which runs
with a fresh context and its own budget.

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

The dividing line: **Claude owns the decision and the verdict; Claude owns the doing.**

If a task is mostly Claude reading its own context to decide the next command, it is
execution-shaped. Delegate it.

## Writing an execution contract

Codex starts with a fresh context. It knows none of this session's reasoning. Everything
it needs must be in the contract — but nothing else. A good contract is short, concrete,
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

# Success criteria
- <objective, checkable> — e.g. "4 result dirs exist, each with metrics.json"

# Failure handling
- On error: fix and retry up to N times.
- If still failing: STOP, write the full error to <path>/BLOCKED.md, report. Do not improvise a workaround.

# Rules
- Do not edit files outside the paths above.
- Do not delete raw data.
- Do not add fallbacks.
- Report: what ran, what the numbers are, what failed.
```

Two rules that matter more than they look:

- **Name the failure path.** Without it, Claude invents workarounds and you get silent
  garbage instead of a clear stop.
- **Make success objective.** "Figure out why it's slow" is not checkable; "4 result dirs
  with metrics.json" is.

Write the contract to a file rather than inlining it — it avoids shell-escaping bugs on
long prompts, and it leaves an artifact you can re-run:

```bash
node "$CODEX_RT" task --write --prompt-file /abs/path/contract.md
```

## Three recipes

Resolve the runtime once per session (the version directory changes on plugin update):

```bash
CODEX_RT="$(ls -d /root/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | tail -1)"
```

### 1. Short, bounded task (< ~10 min) — run it in the foreground

```bash
node "$CODEX_RT" task --write --prompt-file /abs/path/contract.md
```

Blocks, then returns the result. Fine when the answer is needed to make the next decision
immediately.

### 2. Long job (GPU runs, sweeps) — background it and leave

```bash
node "$CODEX_RT" task --background --write --prompt-file /abs/path/contract.md
```

Returns immediately:
```
<title> started in the background as <jobId>. Check /codex:status <jobId> for progress.
```

**Record the jobId.** Then *stop working on that task in this session*. Do not poll in a
loop — each poll is a full turn at full context price, which is the exact cost this skill
exists to avoid. Move to a different phase, or tell the user what is running and end the
turn.

To collect later (a fresh turn, ideally a fresh session):

```bash
node "$CODEX_RT" status <jobId>      # progress
node "$CODEX_RT" result <jobId>      # full output
node "$CODEX_RT" cancel <jobId>      # stop it
```

`result` can be large. Do not paste it wholesale into the main context — read the summary
and the output files it names. Claude Code's own `/codex:status` and `/codex:result`
commands do the same thing if you prefer slash syntax.

### 3. Follow-up fix on the same work — resume, don't restart

```bash
node "$CODEX_RT" task --write --resume-last --prompt-file /abs/path/fix.md
```

`--resume-last` continues Codex's prior context. Use it for "the third instance is wrong,
fix it", "now render the multi-view visualization", "rerun with 1024". Restarting from
scratch throws away everything Claude already learned about the codebase.

Add `--effort high|xhigh` for genuinely hard diagnosis. Leave `--model` alone unless the
user names one.

## Context discipline — the other half of the cost

Delegation cuts turns. Session boundaries cut context size. You need both.

- **Start a new session per phase.** Reading a report, surveying literature, running an
  experiment, and building a viewer are four unrelated topics; carrying them in one
  context means every turn pays for all four.
- **Before switching, write the conclusions down.** Put them in `MEMORY.md` (or a
  findings file) so the next session starts informed instead of re-deriving.
- **Signal to the user.** At the end of a phase, say what is done and suggest `/clear`.
  Do not silently carry on.
- **Rough threshold:** past ~150K context, or past ~40 assistant turns on one topic, the
  next phase belongs in a new session. Cost is superlinear in practice because every
  subsequent turn re-reads the whole thing.

## Anti-patterns

- **Delegating the decision.** "Which of these four failure modes should I chase?" is
  Claude's job. Claude can investigate one you already chose.
- **Delegating a one-liner.** `git status`, `ls`, reading one file — just do it. The
  contract overhead exceeds the work.
- **Polling in the main session.** `status` in a loop costs a full turn each time.
- **Omitting the failure path.** Claude will improvise; you get a plausible wrong answer
  instead of an honest stop.
- **Resuming when the topic changed.** `--resume-last` carries stale assumptions. Use
  `--fresh` (or omit) for new work.
- **Delegating then re-doing it yourself.** Once it is delegated, either use the result or
  cancel it. Doing both pays twice.

## Reporting

The user's global rules require telling them after any command runs, including ones Codex
ran. When reporting a delegated task, state: the jobId, what Claude was told to do, where
output lands, and — once collected — what the numbers actually were. Do not paraphrase a
result you have not read.