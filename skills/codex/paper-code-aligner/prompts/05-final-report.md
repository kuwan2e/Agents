# Prompt 5 - Final Report

Use this checklist immediately before writing `<repo>/paper-code-alignment.md`.

## Required Sections

```markdown
# Paper-Code Alignment Report

## Metadata

## Pipeline Alignment

## Formula Alignment

## Module Notes

## Missing or Ambiguous Mappings

## Paper-Code Drift

## Search Notes
```

## Metadata Must Include

- Repository absolute path.
- Git commit hash.
- Git status summary if dirty.
- PDF absolute path.
- PDF extraction method and command.
- Date.

## Table Requirements

Pipeline table columns:

```text
ID | Paper item | Paper anchor | Code location | Evidence | Status | Confidence
```

Formula table columns:

```text
ID | Formula/objective | Paper anchor | Code location | Evidence | Status | Confidence
```

Code location must include file and nearby line number. Use `Not found` only
for `MISSING` rows.

## Missing or Ambiguous Mappings

For every missing or ambiguous item, include:

- What was searched.
- Which candidate files were checked.
- Why the mapping could not be proven.
- What user/manual evidence would resolve it.

## Paper-Code Drift

For every drift item, include:

- What the paper says.
- What the code does.
- Exact code location.
- Why the difference matters.

## Final Checks

- Every extracted pipeline item has a row.
- Every extracted formula has a row.
- Every `FOUND`, `PARTIAL`, `AMBIGUOUS`, or `DRIFT` row has a code location.
- Every `MISSING` row has search notes.
- The report is written directly to the repository root.
- No source file was changed while performing the audit.
