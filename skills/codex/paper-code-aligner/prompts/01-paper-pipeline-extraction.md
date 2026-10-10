# Prompt 1 - Paper Pipeline Extraction

Use this checklist after extracting text from the PDF. The goal is to build an
internal inventory of paper items that must later be mapped to code.

## Extract These Items

1. Pipeline stages from the method overview, architecture figure, algorithm
   boxes, and experiment setup.
2. Named modules, blocks, networks, heads, encoders, decoders, adapters,
   samplers, retrievers, controllers, or scoring components.
3. Equations and formulas:
   - Losses and objectives.
   - Probability definitions.
   - Update rules.
   - Scoring/ranking functions.
   - Normalization, weighting, or aggregation formulas.
   - Metrics used for reported results.
4. Training procedure:
   - Inputs and targets.
   - Batch construction.
   - Optimizer/scheduler if described.
   - Sampling strategy.
   - Checkpoint/model selection.
5. Inference procedure:
   - Input preprocessing.
   - Forward pass or generation.
   - Decoding/postprocessing.
   - Thresholds, top-k/top-p, beam size, stopping rules.
6. Evaluation procedure:
   - Dataset split.
   - Metrics.
   - Aggregation.
   - Filtering or invalid-output handling.
7. Dataset/preprocessing:
   - Cleaning, resizing, tokenization, normalization.
   - Augmentation.
   - Feature extraction.
   - Exclusions or filtering rules.

## Internal Inventory Format

Use IDs that are stable within the session:

```text
P1, P2, ...  pipeline stages
M1, M2, ...  modules/components
F1, F2, ...  formulas/objectives
A1, A2, ...  algorithms/procedures
H1, H2, ...  hyperparameters/constants
```

For each item record:

```text
ID:
Paper item:
Paper anchor: section/page/equation/figure/table if available
Paper wording: short quote or paraphrase from extracted text
Likely code terms: exact names, symbols, abbreviations, aliases
Must map?: yes/no
```

## Cautions

- PDF extraction can mangle equations. Preserve both symbolic fragments and
  surrounding prose.
- Architecture figure captions often contain module names that do not appear in
  the main prose.
- Appendix sections often contain implementation details absent from the method
  section.
- Do not drop a formula because it looks standard; the user asked for every
  module and formula location.
