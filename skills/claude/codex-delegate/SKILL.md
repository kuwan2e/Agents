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

### 2. Long job (GPU runs, sweeps) — background it and leave

```bash
cd /abs/path && nohup codex exec -C /abs/path --skip-git-repo-check \
  --output-last-message /abs/path/outputs/<name>/CODEX_REPLY.md \
  - < /abs/path/contract.md > /abs/path/outputs/<name>/codex.log 2>&1 &
```

Record the PID. Then *stop working on that task in this session*. Do not poll in a loop —
each poll is a full turn at full context price, which is the exact cost this skill exists
to avoid. Move to another phase, or tell the user what is running and end the turn.

To collect later:

```bash
tail -40 /abs/path/outputs/<name>/codex.log     # progress
cat /abs/path/outputs/<name>/CODEX_REPLY.md     # Codex's own summary
```

Read those two files. Do not paste the whole log into context.

### 3. Follow-up fix on the same work — resume, don't restart

```bash
codex exec resume --last "The third instance is wrong — <specific fix>" \
  -C /abs/path --output-last-message /abs/path/outputs/<name>/CODEX_REPLY.md
```

Resume keeps Codex's accumulated context on the codebase. Use it for "rerun with 1024",
"now render the multi-view visualization", "apply the top fix". Restarting from scratch
throws away everything Codex already learned.

Unrelated new work: start a fresh `codex exec` (do not pass `resume`).

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
- **Pasting the whole log back.** Read the tail and `CODEX_REPLY.md`.
- **Omitting the failure path.** Codex will improvise; you get a plausible wrong answer
  instead of an honest stop.
- **Resuming when the topic changed.** `--last` carries stale assumptions.
- **Delegating then re-doing it yourself.** Either use the result or kill the job. Doing
  both pays twice.
- **Routing through `codex-companion.mjs` / `/codex:rescue`.** See above — it forces the
  bwrap sandbox and fails.

## Reporting

The user's global rules require telling them after any command runs, including those Codex
ran. When reporting a delegated task, state: the PID, what Codex was told to do, where
output lands, and — once collected — what the numbers actually were. Do not paraphrase a
result you have not read.