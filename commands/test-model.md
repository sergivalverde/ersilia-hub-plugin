---
description: Run local tests on an eos-template model before publishing
argument-hint: <model-dir> [--level inspect|shallow]
allowed-tools: [Bash, Read, Glob, Grep, Task, TodoWrite, AskUserQuestion]
---

# Test Model Locally

You are testing a locally generated eos-template model to validate it before publishing to the Ersilia Model Hub.

This command reimplements the relevant checks from `ersilia test` directly, without requiring the ersilia CLI or online model ID validation. This allows testing models with placeholder IDs (like `eos0xxx`) before they are published.

## Parse Arguments

From the user's input, extract:
- `model-dir` (required): Path to the local eos-template model directory (e.g., `./eos0xxx`)
- `--level <level>` (optional): Test depth. `inspect` (metadata + files only) or `shallow` (inspect + end-to-end run). Default: `shallow`

## Phase 1: Inspect Checks

These mirror the "basic" checks from `ersilia test --inspect`.

### 1a. File existence

Check that all mandatory files are present:
```
model/framework/code/main.py
model/framework/run.sh
model/framework/examples/run_input.csv
model/framework/examples/run_output.csv
model/framework/columns/run_columns.csv
install.yml
metadata.yml
```

Report each file as PASS/FAIL. If any are missing, stop.

### 1b. Metadata validation

Read `metadata.yml` and validate each field against the same rules used by `ersilia test` (from `BaseInformationValidator`):

**Required pre-publish fields** (check all of these):

| Field | Validation Rule |
|-------|----------------|
| Identifier | Matches pattern `eos[0-9][a-z0-9]{3}` |
| Slug | Lowercase, 5-60 characters |
| Status | One of: Test, In maintenance, In progress, Ready, Archived |
| Title | 10-300 characters |
| Description | At least 200 characters, non-empty |
| Task | One of: Representation, Annotation, Sampling |
| Subtask | One of: Featurization, Projection, Property calculation or prediction, Activity prediction, Similarity search, Generation |
| Input | Each value one of: Compound, Protein, Text |
| Input Dimension | Integer >= 1 |
| Output | Each value one of: Compound, Score, Value, Text |
| Output Dimension | Integer >= 1 |
| Output Consistency | One of: Fixed, Variable |
| Interpretation | 10-300 characters |
| Tag | Each value from the valid Tag vocabulary (see eos-template-knowledge skill) |
| Biomedical Area | Each value from the valid Biomedical Area vocabulary |
| Target Organism | Each value from the valid Target Organism vocabulary |
| Source | One of: Local, Online |
| Source Type | One of: External, Replicated, Internal |
| Publication | Valid URL or empty |
| Source Code | Valid URL or empty |
| License | Valid SPDX identifier from the vocabulary |
| Publication Type | One of: Peer reviewed, Preprint, Other |

**Post-deployment fields** (skip these — they are filled after publish):
Contributor, DockerHub, S3, Docker Architecture, Image Size, Environment Size, Model Size, Computational Performance, Incorporation Date, Release, Last Packaging Date.

Report each field check as PASS/FAIL.

### 1c. Column and output consistency

1. Read `model/framework/columns/run_columns.csv` — extract the `name` column (skip header row, take first column of each data row)
2. Read `model/framework/examples/run_output.csv` — extract header columns
3. Verify they match exactly (same names, same order, same count)
4. Verify `run_output.csv` has exactly 3 data rows
5. Verify `run_input.csv` has exactly 3 data rows with header `smiles`
6. Verify no `smiles`, `input`, or `key` column appears in the output headers

### 1d. Dependency pinning check

Read `install.yml` and verify:
1. Python version is specified and is one of: 3.8, 3.9, 3.10, 3.11, 3.12
2. Every pip/conda dependency has a pinned version (not unpinned)
3. The file is valid YAML

### 1e. Syntax validation

Verify `main.py` is syntactically valid Python:
```bash
python3 -c "import ast; ast.parse(open('<model-dir>/model/framework/code/main.py').read())"
```

### 1f. Directory size

Report the total size of the model directory and flag any files > 50MB (Git LFS warning).

After all inspect checks, report a summary table. If `--level inspect`, stop here.

## Phase 2: Shallow Checks (end-to-end run)

These mirror the "surface" and "shallow" checks from `ersilia test`, but run main.py directly via bash instead of going through `ersilia fetch/serve/run`.

### 2a. Set up test environment

Create an isolated virtual environment and install the model's dependencies:

```bash
python3 -m venv /tmp/ersilia-test-<model-id>
```

Read `install.yml` and install dependencies:
- For pip entries `["pip", "package", "version"]`: run `/tmp/ersilia-test-<model-id>/bin/pip install package==version`
- For pip entries with index URL: add `--index-url <url>`
- For pip entries with git URL: run `/tmp/ersilia-test-<model-id>/bin/pip install git+<url>`
- For conda entries: warn that conda dependencies need manual setup, but try pip alternatives first (e.g., `rdkit` is pip-installable)
- For string commands: run them in the venv

### 2b. Run main.py on example input

Execute main.py directly, bypassing ersilia serve:

```bash
/tmp/ersilia-test-<model-id>/bin/python3 <model-dir>/model/framework/code/main.py \
    <model-dir>/model/framework/examples/run_input.csv \
    /tmp/ersilia-test-output-<model-id>.csv
```

### 2c. Validate output

1. **Output file exists**: Verify `/tmp/ersilia-test-output-<model-id>.csv` was created
2. **Row count**: Verify output has exactly 3 data rows (matching 3 input SMILES)
3. **Column headers match**: Verify output headers match `run_columns.csv` names
4. **No input columns**: Verify no `smiles`, `input`, or `key` column in output
5. **Values match example**: Compare output against `run_output.csv` — for Fixed consistency, values must match exactly
6. **No invalid values**: Check for NaN, None, null, empty strings in output

### 2d. Consistency check (Fixed output only)

If `Output Consistency` is "Fixed", run main.py a second time and verify outputs are identical:

```bash
/tmp/ersilia-test-<model-id>/bin/python3 <model-dir>/model/framework/code/main.py \
    <model-dir>/model/framework/examples/run_input.csv \
    /tmp/ersilia-test-output-<model-id>-run2.csv
```

Compare both outputs:
- For numeric columns: RMSE must be < 10%
- For string columns: fuzzy match >= 95%
- Report as PASS/FAIL

### 2e. Cleanup

```bash
rm -rf /tmp/ersilia-test-<model-id> /tmp/ersilia-test-output-<model-id>*.csv
```

## Phase 3: Report

Present all results in a clear summary:

```
## Test Results: <model-id>

### Inspect Checks
| Check | Result | Details |
|-------|--------|---------|
| File: main.py | PASS | exists |
| File: run.sh | PASS | exists |
| ... | ... | ... |
| Metadata: Identifier | PASS | eos0xxx |
| Metadata: Title | PASS | 54 chars |
| ... | ... | ... |
| Columns match | PASS | 39 columns |
| Dependencies pinned | PASS | 4 packages |
| Syntax: main.py | PASS | valid Python |

### Shallow Checks
| Check | Result | Details |
|-------|--------|---------|
| Environment setup | PASS | venv created |
| main.py execution | PASS | completed |
| Output rows | PASS | 3 rows |
| Output columns | PASS | match run_columns.csv |
| Output values | PASS | match run_output.csv |
| Consistency | PASS | identical on re-run |

Overall: PASS (X/Y checks passed)
```

For any failed checks, explain what went wrong and suggest fixes.

## Important Rules

- Never modify the model files during testing. This command is read-only.
- The end-to-end run uses main.py directly — it does NOT require ersilia CLI, conda, or Docker.
- Skip post-deployment metadata fields (Contributor, DockerHub, S3, etc.) — those are filled after publish.
- If environment setup fails, report the error but still report all inspect-level results.
- Always clean up test artifacts when done.
