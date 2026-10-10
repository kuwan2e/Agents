# Prompt 4 - Formula Alignment

Use this checklist to map paper formulas, losses, objectives, metrics, and
update rules to code.

## Search Strategy

For each formula:

1. Search the formula name, if any.
2. Search variables and constants from the equation.
3. Search operation names:
   - `loss`, `criterion`, `objective`
   - `score`, `metric`, `rank`, `similarity`
   - `normalize`, `softmax`, `logsumexp`, `entropy`
   - `mean`, `sum`, `weight`, `lambda`, `alpha`, `beta`, `gamma`
4. Search config values and hyperparameters mentioned near the formula.
5. Read candidate implementation with line numbers.

## What Counts as Implemented

A formula can be implemented by:

- A direct expression matching the paper.
- A vectorized or batched equivalent.
- A framework loss function with parameters matching the paper.
- A composed helper function whose call path implements the formula.
- A metric library call with matching options.

The report must explain the equivalence when implementation is not visually
identical to the formula.

## Drift Checks

Check for:

- Sign differences, e.g. maximizing in paper but minimizing negative value.
- Averaging vs summing.
- Different normalization axis.
- Missing weights or extra weights.
- Different epsilon/clamp/log base.
- Different reduction mode.
- Detached gradients where the paper implies end-to-end optimization.
- Constants or thresholds that differ from the paper.

## Per-Formula Notes

Record:

```text
ID:
Formula/objective:
Paper anchor:
Candidate code:
Mathematical equivalence:
Differences:
Status:
Confidence:
Search notes:
```

## Cautions

- PDF text often corrupts symbols. Use nearby prose, equation numbers, and
  variable definitions.
- Standard losses still need code locations.
- If a library function implements the formula, cite the local call site and
  parameters, not external documentation.
