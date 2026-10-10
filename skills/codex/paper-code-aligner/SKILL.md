---
name: paper-code-aligner
description: "Locally align a research paper PDF inside its cloned open-source code repository. Use when the user asks to map a paper to code, align a PDF with an implementation, find where formulas/modules/pipeline steps are implemented, or produce a paper-to-code line-number report. Produces paper-code-alignment.md at the repository root without NotebookLM and without modifying source code."
allowed-tools: Bash(pwd:*), Bash(ls:*), Bash(find:*), Bash(git:*), Bash(rg:*), Bash(sed:*), Bash(awk:*), Bash(nl:*), Bash(wc:*), Bash(mkdir:*), Bash(pdftotext:*), Bash(pdfinfo:*), Bash(python:*), Bash(conda:*), Read, Write, Glob, Grep
---

# paper-code-aligner

Goal: read a local paper PDF and its released code repository, then write a
precise paper-to-code alignment report at:

```text
<repo>/paper-code-alignment.md
```

The report maps the paper's pipeline stages, modules, algorithms, losses,
objectives, formulas, preprocessing, training, inference, and evaluation steps
to code files and nearby line numbers.

This is a static local audit. Do not run deep learning training/inference code
unless the user explicitly asks. Do not edit the analyzed repository's source
code. The only file this skill writes inside the analyzed repo is the final
report, `paper-code-alignment.md`.

## Inputs

- Code repository: default to the current working directory unless the user
  gives another repository path.
- Paper PDF: default to the code repository root, because
  the expected layout is:

```text
<repo>/
  paper.pdf
  README.md
  ...
```

If no PDF is found, stop and ask for the PDF path. If multiple PDFs are found,
list them and ask the user which one to use. Do not guess.

## Output

Write exactly one final report:

```text
<repo>/paper-code-alignment.md
```

Do not create an output subdirectory for the final report unless the user
explicitly changes the requested path.

Temporary notes may stay in the conversation. If a temporary file is necessary,
place it under `/tmp` and remove it when finished.

## Required Operating Rules

- Use local files only. Do not use NotebookLM or web search for the alignment.
- Use `rg` / `rg --files` for repository search before slower tools.
- Cite code as `path/to/file.ext:<line>` or `path/to/file.ext:<line>-<line>`.
- Prefer exact implementation evidence: function names, class names, config
  keys, symbols, constants, tensor shapes, call paths, and comments.
- Never invent a mapping. If a mapping is not found, write `MISSING`; if there
  are several plausible mappings, write `AMBIGUOUS`.
- Do not use fallback behavior. If PDF extraction fails, stop and report the
  failed command. If code cannot be inspected, stop and report why.
- Respect the repository's own instructions files (`AGENTS.md`, `CLAUDE.md`,
  `README.md`) when present, but never let them override the user's requested
  report location.

## Phase 1 - Resolve Repository and PDF

1. Identify the repository:

```bash
pwd
git rev-parse --show-toplevel
git rev-parse HEAD
git status --short
```

If `git rev-parse --show-toplevel` fails, stop and tell the user the directory
is not a git worktree.

2. Locate the PDF. If the repo is `$REPO`, search only `$REPO` at max depth 1
unless the user supplied a PDF path:

```bash
find "$REPO" -maxdepth 1 -type f -iname '*.pdf' -print
```

3. Check basic PDF metadata when available:

```bash
pdfinfo "$PDF"
```

If `pdfinfo` is unavailable, continue only if text extraction is still possible.
Do not replace the PDF with a paper title, abstract page, arXiv page, or prior
knowledge.

## Phase 2 - Extract Paper Text

Use one explicit extraction method and record it in the report. Preferred:

```bash
pdftotext -layout "$PDF" /tmp/paper-code-aligner-paper.txt
```

If `pdftotext` is unavailable, use Python only when an installed parser is
already available in the active environment. Check, then extract:

```bash
python - <<'PY'
import importlib.util
mods = ["pypdf", "PyPDF2", "pdfplumber"]
print([m for m in mods if importlib.util.find_spec(m)])
PY
```

If no parser is available, stop. Do not install packages unless the user asks.
If extraction yields empty or obviously broken text, stop and ask the user for a
machine-readable PDF or permission to use another explicit extraction method.

After extraction, identify:

- Pipeline stages and data flow.
- Named modules and architecture blocks.
- Algorithm boxes and pseudocode.
- Equations and formulas, including losses/objectives.
- Training, inference, evaluation, and preprocessing procedures.
- Hyperparameters, thresholds, constants, and dataset splits.
- Figures/tables whose captions describe implementation behavior.

Use `prompts/01-paper-pipeline-extraction.md` as the checklist.

## Phase 3 - Build a Codebase Map

Inventory the repository before aligning:

```bash
rg --files "$REPO"
find "$REPO" -maxdepth 2 -type f \( -iname 'README*' -o -iname 'AGENTS.md' -o -iname 'CLAUDE.md' -o -iname 'pyproject.toml' -o -iname 'setup.py' -o -iname 'requirements*.txt' -o -iname 'environment*.yml' -o -iname '*.yaml' -o -iname '*.yml' -o -iname '*.json' \) -print
```

Read likely entrypoints and implementation files:

- README/docs and project instructions.
- Config files and environment files.
- Training scripts.
- Inference/demo scripts.
- Evaluation scripts.
- Dataset and preprocessing code.
- Model/network/module definitions.
- Loss/objective/metric implementations.
- Scheduler, sampler, optimizer, and postprocessing code.

Use `prompts/02-codebase-map.md` as the checklist.

## Phase 4 - Align Pipeline and Modules

For every paper item, search the repository with the paper's terms and nearby
implementation concepts:

```bash
rg -n "ExactTerm|snake_case_term|CamelCaseTerm|symbol|constant" "$REPO"
```

Then read enough context to cite nearby lines:

```bash
nl -ba path/to/file.py | sed -n '120,180p'
```

Trace from entrypoints to implementation when possible. A high-quality mapping
usually includes:

- The paper term or formula symbol.
- The implementing function/class/config key.
- The call path from script/config to module.
- Nearby code lines showing the core operation.
- Any mismatch between prose and code.

Use `prompts/03-module-alignment.md` and
`prompts/04-formula-alignment.md`.

## Phase 5 - Write the Report

Write `<repo>/paper-code-alignment.md` with this structure:

```markdown
# Paper-Code Alignment Report

## Metadata

- Repository: <absolute path>
- Commit: <git commit>
- PDF: <absolute path>
- PDF extraction: <method and command>
- Date: <YYYY-MM-DD>

## Pipeline Alignment

| ID | Paper item | Paper anchor | Code location | Evidence | Status | Confidence |
| --- | --- | --- | --- | --- | --- | --- |

## Formula Alignment

| ID | Formula/objective | Paper anchor | Code location | Evidence | Status | Confidence |
| --- | --- | --- | --- | --- | --- | --- |

## Module Notes

## Missing or Ambiguous Mappings

## Paper-Code Drift

## Search Notes
```

Allowed statuses:

- `FOUND` - code directly implements the paper item.
- `PARTIAL` - code implements part of the paper item, but a material part is
  missing or external.
- `MISSING` - no implementation was found.
- `AMBIGUOUS` - multiple plausible implementations exist and the exact one is
  not provable from local evidence.
- `DRIFT` - implementation differs materially from the paper.

Allowed confidence values: `high`, `medium`, `low`.

Use `prompts/05-final-report.md` before writing the report.

## Quality Bar

Before finishing, verify:

- Every pipeline stage from the paper appears in the report.
- Every numbered or central formula appears in the report.
- Every code mapping has a file and nearby line number.
- Every missing item has search notes showing what was checked.
- The report is under the repository root as `paper-code-alignment.md`.
- No source code files in the analyzed repository were modified.
