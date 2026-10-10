# Prompt 3 - Module and Pipeline Alignment

Use this checklist to map paper pipeline stages and modules to source code.

## Alignment Procedure

For each paper item:

1. Search exact terms from the paper.
2. Search likely code variants:
   - lowercase / snake_case / CamelCase
   - abbreviation
   - symbol name
   - config key
   - class/function suffixes such as `Module`, `Layer`, `Head`, `Loss`
3. Read the best candidate files with line numbers.
4. Trace callers and callees when the implementation is split.
5. Decide status and confidence.

## Evidence That Supports a Mapping

Strong evidence:

- Same distinctive module name.
- Same equation variables or constants.
- Same ordered steps as the paper pipeline.
- Same config keys and default values.
- Entrypoint calls the candidate implementation on the relevant path.
- README command or script name ties the file to the paper's experiment.

Weak evidence:

- Generic class/function names without paper-specific terms.
- Similar operation but no call path.
- Same concept implemented in multiple places.
- Code appears unused by training/evaluation scripts.

## Status Rules

- `FOUND`: direct implementation with clear evidence and nearby line numbers.
- `PARTIAL`: implementation covers only part of the paper item.
- `MISSING`: targeted searches found no implementation.
- `AMBIGUOUS`: multiple plausible implementations remain.
- `DRIFT`: code behavior materially differs from the paper.

## Per-Item Notes

Record:

```text
ID:
Paper item:
Candidate code:
Call path:
Evidence:
Differences:
Status:
Confidence:
Search notes:
```

## Cautions

- Do not map a paper item to a file solely because names are similar.
- Do not ignore preprocessing, postprocessing, and evaluation aggregation.
- If code has several versions of a module, report the ambiguity unless the
  entrypoint or config proves which one is used.
