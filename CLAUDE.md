## Rules
- Always download checkpoints to /workspace/checkpoints
- Always use conda to manage Python environments. One project One environment. Always check if there are some submodules, use git.
- Always install DeepLearning tools(pytorch, tensorflow ...) with cuda. Note that default cuda version is 12.4. You can also switch to /workspace/cuda-11.8.
- When preprocess dataset, do NOT delete any raw data of dataset.
- When Configure Python environment and run a demo, do NOT edit any raw code of project. But you can edit after I allow.
- All Results/Outputs should be saved to a sub folder of project path.
- Remember you can use 4x A100 GPUs. Consider multi-gpu Training/Inference.
- Before run code, check conda environments to find if there is an existing env for code. Also check if I have accessible GPUs. If not, tell me. Do not run any deeplearning code with cpu.
- When I want to download something, just download it whatever the size it is. And remember to use nohup with log.
- Always tell me after you use any command include python, bash, etc...
- Always use offline rendering. There is no GUI on this server.
- Never edit files outside the project path, except files in directories pointed to by symbolic links within the project path.
- Do NOT use fallback.
- Update the git version every time the code changes more than 100 lines. Use git commit.
- When I don't specifically emphasize which directory's code to run, we now run the code of whichever project we are under, regardless of the data path.

## Delegating execution to Codex

My expensive resource is not output volume, it is **assistant turn count × current context size**. Every turn costs the full context, so the fix is fewer turns in a big context — not shorter answers.

**Delegate to Codex (use the `codex-delegate` skill for the full workflow) whenever the task is:**
- running an experiment, training, or inference
- writing a script and then iterating on it until it runs
- building an HTML/JS visualization or viewer
- environment setup, dependency/conda work
- bulk reruns, sweeps, multi-seed or multi-config launches
- reading logs to diagnose why a run failed
- any refactor spanning many files

**Keep it yourself:** deciding *what* to run and *why*, interpreting results, judging whether a difference matters, priorities and tradeoffs, planning, and anything requiring my input.

Invoke Codex with `codex exec` directly. **Do not use `/codex:rescue` or `codex-companion.mjs`** — they hardcode the `workspace-write` sandbox, which needs bubblewrap and fails here with `bwrap: No permissions to create a new namespace`. Plain `codex exec` reads my `~/.codex/config.toml` (`sandbox_mode = "danger-full-access"`) and never touches bwrap. This is verified working.

Before delegating, write an **execution contract** to a file: goal, code root, inputs, output paths, conda env, GPUs, steps, objective success criteria, and an explicit failure path ("on error, retry N times, then stop and write BLOCKED.md — do not improvise"). Codex starts fresh and has none of this session's reasoning, but it also has none of this session's bloat.

**Every code change requires a Codex handoff and Claude review.** Include the intended design, allowed scope, acceptance criteria, and handoff requirements in each contract, including follow-up fixes. Codex must record the pre-edit baseline and existing user changes, then report in `CODEX_REPLY.md`: every added/modified/deleted file with relevant symbols or lines; what changed, why, and which requirement it implements; how to locate the complete diff, including committed and new files; commands/checks and actual results; checks not run, deviations, risks, and blockers. If no code changed, say so explicitly.

**Claude must inspect the actual diff and relevant surrounding code before accepting the result.** Compare it with the delegated task's design, scope, and acceptance criteria; cover committed, staged, unstaged, and new files without attributing pre-existing user edits to Codex. A Codex summary, successful exit, or passing check alone is not approval. Use only necessary focused validation. Record Claude's own verdict (`PASS`, `NEEDS_CHANGES`, or `BLOCKED`), reviewed revision/diff, evidence, limits, and file/line findings in the task's `CLAUDE_REVIEW.md`.

**Review failures go back to Codex for correction and another review.** Resume the specific task session with concrete findings and expected behavior, require an updated change report, and review again after every code-changing repair. Stop at the contract's retry limit or a blocker; report any needed design decision instead of silently changing the design or acceptance criteria. Only Claude's `PASS` on the current code permits reporting the task complete; otherwise state that it is awaiting review, needs changes, or is blocked. Tell the user what changed and the review/validation result.

Long jobs: launch under `nohup` with a log file, record the PID, and **do not poll in the main conversation**. Immediately attach a one-shot watcher using `Bash` with `run_in_background: true`; wait for the exact job to exit and report its exit status. Treat `CODEX_REPLY.md` as a reply, not proof of task success; read `BLOCKED.md`, logs, and check the contract's acceptance criteria. The watcher is session-scoped and best-effort, so keep durable job metadata and reconcile outstanding jobs when a new session starts. Always pass `--output-last-message <FILE>` and redirect stdout/stderr to a log, so only a compact result enters the main context. Follow-up fixes resume the specific Codex session ID; unrelated new work starts a fresh `codex exec`. Never bypass a denied tool action through a delegate.

**Session boundaries matter as much as delegation.** Start a new session per phase — reading a report, surveying literature, running experiments, and building a viewer are unrelated topics, and carrying them together makes every turn pay for all of them. Before switching, write the conclusions into `MEMORY.md`, then tell me the phase is done and suggest `/clear`. Past roughly 150K context, or past ~40 assistant turns on one topic, the next phase belongs in a fresh session.

Note: `/root/.codex/AGENTS.md` already contains a "Do NOT do too many test. I need experiment, not test." rule — keep that file in sync when these rules change.

