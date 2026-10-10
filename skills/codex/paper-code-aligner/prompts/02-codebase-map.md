# Prompt 2 - Codebase Map

Use this checklist before trying to align individual paper items. The goal is
to understand where implementation concepts live.

## Repository Inventory

Identify:

1. Project type and framework.
2. Main entrypoints:
   - training
   - inference/demo
   - evaluation
   - preprocessing/data preparation
3. Configuration system:
   - YAML/JSON/TOML files
   - command-line arguments
   - dataclasses or config objects
4. Core source directories.
5. External submodules or vendored code.
6. Generated artifacts, checkpoints, or datasets that should not be treated as
   source implementation.

## Files to Prioritize

Read these first when present:

- `README*`, `docs/*`, `AGENTS.md`, `CLAUDE.md`.
- `train*`, `main*`, `run*`, `infer*`, `demo*`, `eval*`, `test*`.
- `model*`, `network*`, `module*`, `layers*`, `backbone*`.
- `loss*`, `objective*`, `criterion*`, `metric*`, `score*`.
- `dataset*`, `data*`, `preprocess*`, `transform*`, `augment*`.
- `config*`, `default*`, `options*`, `args*`.

## Map Format

Build an internal map:

```text
Role:
Files:
Important symbols:
Entrypoints/callers:
Notes:
```

Useful roles:

- training entrypoint
- inference entrypoint
- evaluation entrypoint
- preprocessing/data loading
- model architecture
- loss/objective
- metric/scoring
- configuration
- postprocessing
- checkpoint/loading

## Cautions

- A README command often points to the real entrypoint.
- The reported paper numbers may come from evaluation scripts rather than the
  model code.
- Config defaults can contradict paper hyperparameters.
- A formula may be implemented under a generic name like `criterion`,
  `forward`, `score`, or `compute_loss`.
