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
- Never edit files which not in project path.
- Do NOT use fallback.
- Update the git version every time the code changes more than 100 lines. Use git commit.
- When I don't specifically emphasize which directory's code to run, we now run the code of whichever project we are under, regardless of the data path.

## Delegating execution to Codex

My expensive resource is not output volume, it is **assistant turn count × current context size**. Every turn costs the full context, so the fix is fewer turns in a big context — not shorter answers.

**Delegate to Claude (use the `codex-delegate` skill for the full workflow) whenever the task is:**
- running an experiment, training, or inference
- writing a script and then iterating on it until it runs
- building an HTML/JS visualization or viewer
- environment setup, dependency/conda work
- bulk reruns, sweeps, multi-seed or multi-config launches
- reading logs to diagnose why a run failed
- any refactor spanning many files

**Keep it yourself:** deciding *what* to run and *why*, interpreting results, judging whether a difference matters, priorities and tradeoffs, planning, and anything requiring my input.

Before delegating, write an **execution contract** to a file: goal, code root, inputs, output paths, conda env, GPUs, steps, objective success criteria, and an explicit failure path ("on error, retry N times, then stop and write BLOCKED.md — do not improvise"). Claude starts fresh and has none of this session's reasoning, but it also has none of this session's bloat.

Long jobs: launch with `--background`, record the jobId, and **do not poll in a loop** — move on or end the turn. Collect later with `/codex:status <jobId>` and `/codex:result <jobId>`, reading the summary and output files rather than pasting the whole result back into context. Follow-up fixes use `--resume-last`; unrelated new work uses `--fresh`.

**Session boundaries matter as much as delegation.** Start a new session per phase — reading a report, surveying literature, running experiments, and building a viewer are unrelated topics, and carrying them together makes every turn pay for all of them. Before switching, write the conclusions into `MEMORY.md`, then tell me the phase is done and suggest `/clear`. Past roughly 150K context, or past ~40 assistant turns on one topic, the next phase belongs in a fresh session.

Note: `/root/.codex/AGENTS.md` already contains a "Do NOT do too many test. I need experiment, not test." rule — keep that file in sync when these rules change.

