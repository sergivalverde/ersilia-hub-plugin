---
description: Run local tests on an eos-template model before publishing
argument-hint: <model-dir> [--level inspect|shallow|deep]
allowed-tools: [Bash, Read, Glob, Grep, Task, TodoWrite, AskUserQuestion]
---

# Test Model Locally

You are testing a locally generated eos-template model to validate it before publishing to the Ersilia Model Hub.

This command reimplements the relevant checks from `ersilia test` directly, without requiring the ersilia CLI or online model ID validation. This allows testing models with placeholder IDs (like `eos0xxx`) before they are published.

## Parse Arguments

From the user's input, extract:
- `model-dir` (required): Path to the local eos-template model directory (e.g., `./eos0xxx`)
- `--level <level>` (optional): Test depth. `inspect` (metadata + files only), `shallow` (inspect + end-to-end run), or `deep` (shallow + scientific validation). Default: `shallow`

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

## Phase 3: Deep Checks (scientific validation)

Only run if `--level deep`. Uses the same venv from Phase 2 (do not clean up yet).

### 3a. Load metadata for routing

Read `metadata.yml` to get Task, Tags, Biomedical Area, Output type, Output Dimension.
Read `run_columns.csv` to get column names, types, and directions.

### 3b. Distribution analysis (all model types)

Write the DIVERSE_50 SMILES list (from the eos-template-knowledge skill) to a temp CSV:

```bash
# Write 50 diverse SMILES to temp input file
python3 -c "
smiles = [<DIVERSE_50 list from skill>]
with open('/tmp/deep-diverse-input-<model-id>.csv', 'w') as f:
    f.write('smiles\n')
    for s in smiles:
        f.write(s + '\n')
"
```

Run main.py on the diverse set and measure runtime:

```bash
time /tmp/ersilia-test-<model-id>/bin/python3 <model-dir>/model/framework/code/main.py \
    /tmp/deep-diverse-input-<model-id>.csv \
    /tmp/deep-diverse-output-<model-id>.csv
```

Parse the output and compute statistics per column:

```python
import csv, math

# For each numeric column, compute:
stats = {
    "count_valid": 0,       # non-null, non-NaN values
    "count_null": 0,        # null/NaN/empty values
    "mean": 0.0,
    "std": 0.0,
    "min": 0.0,
    "max": 0.0,
    "coefficient_of_variation": 0.0,  # std/mean if mean != 0
    "is_constant": False,   # True if std == 0
    "all_finite": True,     # False if any inf
}
```

**Check criteria**:

| Check | PASS | WARNING | FAIL |
|-------|------|---------|------|
| Completion rate | >= 45/50 (90%) | 35-44/50 (70-88%) | < 35/50 (< 70%) |
| Non-constant | std > 1e-6 for each column | — | Any column has std = 0 |
| Finite values | All finite | — | Any inf value |
| Runtime | < 300 seconds | 300-600 seconds | > 600 seconds |

**Additional checks by model type**:

For **Representation** models (fingerprints/embeddings):
- Count how many columns have std > 0 across the 50 molecules
- PASS if > 50% of columns vary; WARNING if 10-50%; FAIL if < 10%

For **Sampling** models (compound generation output):
- Parse each output value as SMILES using a regex heuristic (contains at least one of: `C`, `c`, `N`, `n`, `O`, `o` and no spaces)
- PASS if >= 80% parse; WARNING if 50-80%; FAIL if < 50%
- Check diversity: count unique output strings, PASS if >= 50% unique

### 3c. Sanity checks (Annotation and Representation models only)

Resolve the model's Tags and Biomedical Area to sanity categories using the TAG_TO_SANITY_KEY and BIOMEDICAL_AREA_TO_SANITY_KEY mappings from the eos-template-knowledge skill.

If Task is `Sampling`, skip sanity checks.
If no categories matched and Task is `Representation`, default to `"fingerprint"`.
If no categories matched at all, skip with WARNING status.

For each matched sanity category:

**For Annotation models** (Output is Score or Value):

1. Write positive compounds to a temp CSV, run main.py, compute mean per output column
2. Write negative compounds to a temp CSV, run main.py, compute mean per output column
3. For each column, read `direction` from `run_columns.csv`:
   - If direction is `high`: check mean(positive) > mean(negative)
   - If direction is `low`: check mean(positive) < mean(negative)
   - If no direction: skip this column
4. PASS if separation is correct; WARNING if difference < 10% of range; FAIL if reversed

For solubility/lipophilicity, use `high_soluble`/`low_soluble` or `high_logp`/`low_logp` as the two groups. Map to positive/negative based on the column's direction.

**For Representation models** (fingerprints/embeddings):

1. Run the `similar_pair` compounds (aspirin + salicylic acid) through main.py
2. Run the `dissimilar_pair` compounds (eicosane + adenine) through main.py
3. Compute cosine similarity between the two output vectors in each pair:
   ```python
   import math
   def cosine_sim(a, b):
       dot = sum(x*y for x,y in zip(a,b))
       norm_a = math.sqrt(sum(x*x for x in a))
       norm_b = math.sqrt(sum(x*x for x in b))
       if norm_a == 0 or norm_b == 0:
           return 0.0
       return dot / (norm_a * norm_b)
   ```
4. PASS if similarity(similar_pair) > similarity(dissimilar_pair); FAIL if reversed

### 3d. Paper reproduction (when Publication URL is available)

Read `Publication` from `metadata.yml`. If empty or null, skip with status `"skipped"`.

**Step 1 — Fetch the paper**:

Use WebFetch to extract structured information from the paper URL:

```
Prompt: "Extract from this scientific paper in JSON format:
{
  \"model_type\": \"classification or regression or generation or embedding\",
  \"reported_metrics\": [{\"metric_name\": \"...\", \"value\": ..., \"dataset\": \"...\"}],
  \"test_smiles_mentioned\": [\"SMILES1\", ...],
  \"key_finding\": \"one sentence summary\"
}
If a field cannot be determined, set it to null."
```

**Step 2 — Look for test data**:

If `Source Code` URL points to a GitHub repo:
1. Check if the source repo was already cloned at `/tmp/ersilia-source-*`
2. Search for CSV/SDF files containing SMILES in `test/`, `data/`, `examples/` directories
3. If found, extract up to 30 SMILES with their labels (if labeled)

**Step 3 — Run and compare** (only if test data with labels was found):

1. Write test SMILES to temp CSV
2. Run main.py
3. Compute the same metric the paper reports (AUC for classification, RMSE/R² for regression)
4. Compare: within 20% relative deviation = PASS; within 50% = WARNING; else WARNING (never FAIL)

**Paper reproduction never produces a hard FAIL** — the eos-template wrapping may legitimately differ from the paper's exact evaluation pipeline.

| Condition | Status |
|-----------|--------|
| Metric reproduced within 20% | passed |
| Metric reproduced within 50% | warning |
| Metric deviation > 50% | warning |
| No paper URL | skipped |
| Paper fetched but no metrics found | skipped |
| No test data available | skipped |

### 3e. Generate deep validation notebook

When `--level deep`, generate an **executed** Jupyter notebook (`deep_validation.ipynb`) in the model directory. This notebook provides an interactive, visual report of all deep validation results with plots and tables.

**Step 1 — Install notebook dependencies in the test venv:**
```bash
/tmp/ersilia-test-<model-id>/bin/pip install matplotlib seaborn pandas ipykernel nbconvert jupyter_client
/tmp/ersilia-test-<model-id>/bin/python3 -m ipykernel install --user --name <model-id>-validation --display-name "Python (<model-id>)"
```

**Step 2 — Generate the notebook source:**

Write a `deep_validation.ipynb` file in `<model-dir>/` with these sections:

1. **Setup**: Import the model's inference library directly (inline, not subprocess). Define `run_model(smiles_list)` returning DataFrame + elapsed time. Define `cosine_similarity(a, b)`.

2. **Distribution Analysis**: Run DIVERSE_50 molecules, show completion stats, fingerprint/output heatmap (varying columns only), per-column mean/std bar chart, pairwise similarity histogram + heatmap.

3. **Sanity Checks**: Run matched sanity compounds. For fingerprint models: similar vs dissimilar pair cosine similarity with side-by-side bar charts. For annotation models: positive vs negative group means with directional bar charts.

4. **Paper Reproduction** (if applicable): Run paper-specific validation with appropriate visualizations (similarity matrices, boxplots, metric comparisons). Show "Skipped" if no paper data.

5. **Summary**: Results table + JSON export matching `deep_checks` schema.

**Step 3 — Execute the notebook:**
```bash
/tmp/ersilia-test-<model-id>/bin/jupyter nbconvert \
    --to notebook --execute \
    --ExecutePreprocessor.kernel_name=<model-id>-validation \
    --ExecutePreprocessor.timeout=600 \
    --output deep_validation.ipynb \
    <model-dir>/deep_validation.ipynb
```

This overwrites the notebook with the executed version containing all outputs and plots. If execution fails, keep the un-executed notebook and note the failure.

**Step 4 — Clean up kernel:**
```bash
jupyter kernelspec remove <model-id>-validation -f 2>/dev/null || true
```

After deep checks and notebook generation, clean up the venv:

```bash
rm -rf /tmp/ersilia-test-<model-id> /tmp/ersilia-test-output-<model-id>*.csv /tmp/deep-*.csv
```

## Phase 4: Report

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

### Deep Checks (if --level deep)
| Check | Result | Details |
|-------|--------|---------|
| Distribution: completeness | PASS | 48/50 valid outputs |
| Distribution: non-constant | PASS | all columns vary |
| Distribution: runtime | PASS | 42s for 50 molecules |
| Sanity (fingerprint) | PASS | similar pair cosine > dissimilar |
| Paper reproduction | SKIPPED | no test data found |

Overall: PASS (X/Y checks passed, W warnings)
```

For any failed checks, explain what went wrong and suggest fixes.
For warnings (especially from paper reproduction), explain what was checked and why it was inconclusive.

If `--level deep` was used, also note whether `deep_validation.ipynb` was generated and executed successfully. The notebook contains interactive plots that can be viewed in the model repository.

## Important Rules

- The only file this command writes to the model directory is `deep_validation.ipynb` (when `--level deep`). All other model files are read-only.
- The end-to-end run uses main.py directly — it does NOT require ersilia CLI, conda, or Docker.
- Skip post-deployment metadata fields (Contributor, DockerHub, S3, etc.) — those are filled after publish.
- If environment setup fails, report the error but still report all inspect-level results.
- Always clean up test artifacts when done.
