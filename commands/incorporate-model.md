---
description: Incorporate a published ML model into the Ersilia Model Hub
argument-hint: <repo-url> [--paper <paper-url>] [--model-id <eosXXXX>] [--output-dir <path>]
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep, WebFetch, Task, AskUserQuestion, TodoWrite]
---

# Incorporate Model into Ersilia Model Hub

You are incorporating a machine learning model into the Ersilia Model Hub by wrapping it in the eos-template format.

## Parse Arguments

From the user's input, extract:
- `repo-url` (required): GitHub URL or Zenodo URL of the source model
- `--paper <url>` (optional): URL of the associated publication
- `--model-id <id>` (optional): Ersilia model identifier (eosXXXX format). If not provided, a unique ID will be generated automatically in Phase 3
- `--output-dir <path>` (optional): Where to create the model directory. Default: current working directory

During Phases 1 and 2, use a temporary working name (e.g., the repo name) for directories. The final model ID is assigned in Phase 3.

## Phase 1: Clone and Analyze

1. **Clone the source repository**:
   ```
   git clone <repo-url> /tmp/ersilia-source-<model-id>
   ```
   For Zenodo URLs, download with `curl -L -o /tmp/model.zip <url>` and extract.

2. **Read everything relevant**:
   - README.md (understand what the model does)
   - Dependency files: requirements.txt, setup.py, pyproject.toml, environment.yml, conda.yml, Pipfile
   - Find the inference/prediction entry point by searching for: predict functions, forward methods, `__main__` blocks, CLI scripts, `if __name__` blocks
   - Identify all checkpoint/weight files and their sizes
   - Determine the ML framework (PyTorch, TensorFlow, scikit-learn, XGBoost, RDKit, Chemprop, custom)
   - Determine input format (single SMILES, SMILES pairs, protein sequences, text)
   - Determine output format (single float, multiple floats, embeddings, SMILES, text)

3. **If a paper URL was provided**, fetch it and extract:
   - What the model predicts and the biological context
   - Training data source and size
   - Key performance metrics

## Phase 2: Verify by Running

This phase is MANDATORY. Do not skip it. The purpose is to confirm that your analysis from Phase 1 is correct by actually executing the model code.

### 2a. Install the model in an isolated environment

Create a temporary virtual environment and install the model's dependencies:

```bash
python3 -m venv /tmp/ersilia-verify-<model-id>
```

Then install the model. Choose the appropriate method:
- **If pip-installable** (has setup.py or pyproject.toml): `/tmp/ersilia-verify-<model-id>/bin/pip install <package-name>` or `/tmp/ersilia-verify-<model-id>/bin/pip install /tmp/ersilia-source-<model-id>`
- **If not pip-installable**: install dependencies manually with `/tmp/ersilia-verify-<model-id>/bin/pip install <dep1> <dep2> ...`
- **If conda is required** (e.g. for rdkit in older versions): note this for install.yml but try pip-based alternatives first (rdkit is now pip-installable)

### 2b. Run the model on test inputs

Execute the model's inference on the 3 default test molecules (aspirin, ibuprofen, caffeine) using the entry point identified in Phase 1:

```bash
/tmp/ersilia-verify-<model-id>/bin/python3 -c "
# Adapt this to the actual model inference code identified in Phase 1
# Example for a fingerprint model:
from model_package import predict
smiles = ['CC(=O)Oc1ccccc1C(=O)O', 'CC(C)Cc1ccc(cc1)C(C)C(=O)O', 'Cn1c(=O)c2c(ncn2C)n(c1=O)C']
for smi in smiles:
    result = predict(smi)
    print(f'SMILES: {smi}')
    print(f'  Output type: {type(result)}')
    print(f'  Output length: {len(result) if hasattr(result, \"__len__\") else 1}')
    print(f'  Output value: {result}')
"
```

### 2c. Verify and correct your analysis

From the actual execution, confirm or correct:

1. **Output dimension**: Count the actual number of output values. Do NOT rely on counting items in source code — the running code is the ground truth.
2. **Output type**: Verify whether outputs are floats, integers, strings, etc.
3. **Output column names**: If the model has named outputs (e.g., substructure keys, property names), extract them programmatically from the running code rather than counting manually.
4. **Input handling**: Confirm the model accepts the expected input format (SMILES, InChI, etc.) and handles invalid inputs gracefully.
5. **Dependencies**: Note the exact versions that were installed (run `/tmp/ersilia-verify-<model-id>/bin/pip freeze | grep <key-packages>`).

### 2d. Record verified example outputs

Save the actual outputs from the test run. These will be used as the ground-truth `run_output.csv` in Phase 4 — do NOT use placeholder values.

### 2e. Handle verification failures

- **If installation fails**: investigate the error, try alternative installation methods, and document what worked. If nothing works, report to the user and stop.
- **If inference fails**: check for missing dependencies, incorrect entry points, or version incompatibilities. Try to fix and re-run. If it cannot be resolved, report the error to the user and stop.
- **If outputs differ from analysis**: update your analysis to match the verified results. The running code is always the source of truth.

## Phase 3: Assign Model ID and Present Verified Analysis

### 3a. Generate or confirm the model ID

If the user provided `--model-id`, use that ID. Otherwise, generate a unique one:

```bash
python3 -c "
import json, random, string, subprocess, os

def encode():
    result = 'eos'
    result += random.choice('123456789')
    result += ''.join(random.choice(string.ascii_lowercase + '0123456789') for _ in range(3))
    return result

def exists(model_id):
    try:
        output = subprocess.check_output(
            ['gh', 'api', f'repos/ersilia-os/{model_id}', '--jq', '.name'],
            stderr=open(os.devnull, 'w')
        )
        return True
    except subprocess.CalledProcessError:
        return False

while True:
    model_id = encode()
    if not exists(model_id):
        print(model_id)
        break
"
```

This generates a random ID matching `eos[1-9][a-z0-9]{3}` and verifies it doesn't already exist as a repo under `ersilia-os`.

### 3b. Present verified analysis with the assigned ID

Present your **verified** analysis to the user, clearly marking what was confirmed by execution:

```
## Verified Model Analysis

**Model ID**: <generated-or-provided-id>
**What it does**: [one paragraph]
**ML Framework**: [framework name]
**Input**: [format description]
**Output**: [format description] (VERIFIED: ran on 3 test molecules)
**Output Dimension**: [number] (VERIFIED: actual output length from test run)
**Dependencies**: [key packages with pinned versions from pip freeze]
**Checkpoint files**: [list with sizes, or "None" for pip-installable models]
**Inference entry point**: [file and function/method] (VERIFIED: successfully called)

## Verified Test Results
| SMILES | Output |
|--------|--------|
| CC(=O)Oc1ccccc1C(=O)O | [actual values] |
| CC(C)Cc1ccc(cc1)C(C)C(=O)O | [actual values] |
| Cn1c(=O)c2c(ncn2C)n(c1=O)C | [actual values] |

## Proposed Metadata
- Task: [from valid vocabulary]
- Subtask: [from valid vocabulary]
- Input: [Compound/Protein/Text]
- Output: [Score/Value/Compound/Text]
- Tags: [from valid vocabulary]
- Biomedical Area: [from valid vocabulary]
- Target Organism: [from valid vocabulary]

## Proposed Output Columns
| name | type | direction | description |
|------|------|-----------|-------------|
| ... | float/integer/string | high/low/ | ... |
```

Then ask the user: "This analysis has been verified by running the model. The assigned model ID is `<model-id>`. Should I proceed with generating the eos-template files, or would you like to adjust anything (including the model ID)?"

Wait for confirmation before proceeding. If the user wants a different ID, re-validate that it matches `eos[1-9][a-z0-9]{3}` and doesn't exist on GitHub.

## Phase 4: Generate Files

After user confirmation, generate all files:

### 4a. Create directory structure
```
mkdir -p <output-dir>/<model-id>/model/checkpoints
mkdir -p <output-dir>/<model-id>/model/framework/code
mkdir -p <output-dir>/<model-id>/model/framework/examples
mkdir -p <output-dir>/<model-id>/model/framework/columns
```

### 4b. Write model/framework/code/main.py

This MUST be functional code, not a placeholder. Follow the main.py contract from the eos-template-knowledge skill:
- `sys.argv[1]` = input CSV, `sys.argv[2]` = output CSV
- Read SMILES from input CSV (skip header, take first column)
- Call the actual model inference
- Assert input length equals output length
- Write output CSV with appropriate headers (matching run_columns.csv)

Copy any source code files from the cloned repo that main.py needs into `model/framework/code/`. Adjust import paths accordingly.

### 4c. Copy checkpoint files
Copy pre-trained weights from the source repo into `model/checkpoints/`.

### 4d. Write model/framework/run.sh
```
python $1/code/main.py $2 $3
```

### 4e. Write install.yml
Pin all dependency versions using the exact versions confirmed during verification (Phase 2e pip freeze output). Use the source repo's dependency files as reference. Prefer Python 3.10 unless the model requires a specific version.

### 4f. Write metadata.yml
Use ONLY values from the valid vocabulary lists in the eos-template-knowledge skill. Set Status to "In progress". Set Source Type to "External" for third-party models.

### 4g. Write example files
- `model/framework/examples/run_input.csv`: 3 SMILES with header `smiles`. Use aspirin (`CC(=O)Oc1ccccc1C(=O)O`), ibuprofen (`CC(C)Cc1ccc(cc1)C(C)C(=O)O`), caffeine (`Cn1c(=O)c2c(ncn2C)n(c1=O)C`) as safe defaults.
- `model/framework/examples/run_output.csv`: Use the ACTUAL outputs recorded during verification (Phase 2d). Never use placeholder values.
- `model/framework/columns/run_columns.csv`: column metadata with name, type, direction, description

## Phase 5: Test and Generate Report

This phase runs the same checks as `/test-model --level deep` and saves a test report file in the model directory as evidence. This report is committed alongside the model files when published.

### 5a. Run all inspect checks

Using the verification venv from Phase 2, run these checks programmatically and collect results as a list of `(check_name, status, detail)` tuples:

**File existence** — verify all 7 mandatory files exist.

**Metadata validation** — validate each pre-publish field against the same rules used by `ersilia test` (from `BaseInformationValidator`):

| Field | Rule |
|-------|------|
| Identifier | Pattern `eos[0-9][a-z0-9]{3}` |
| Slug | Lowercase, 5-60 chars |
| Status | Valid vocabulary |
| Title | 10-300 chars |
| Description | >= 200 chars |
| Task, Subtask, Input, Output, Output Consistency | Valid vocabulary |
| Input Dimension, Output Dimension | Integer >= 1 |
| Interpretation | 10-300 chars |
| Tag, Biomedical Area, Target Organism | Valid vocabulary lists |
| Source, Source Type | Valid vocabulary |
| Publication, Source Code | Valid URL or empty |
| License, Publication Type | Valid vocabulary |

Skip post-deployment fields (Contributor, DockerHub, S3, Docker Architecture, Image Size, etc.).

**Column consistency** — verify run_columns.csv names match run_output.csv headers, 3 rows each, no smiles/input/key in output.

**Dependency pinning** — verify install.yml has valid Python version and all deps pinned.

**Syntax** — verify main.py parses as valid Python.

### 5b. Run end-to-end checks

Using the verification venv from Phase 2:

1. **Run main.py** on the example input:
   ```bash
   /tmp/ersilia-verify-<model-id>/bin/python3 <path>/model/framework/code/main.py \
       <path>/model/framework/examples/run_input.csv \
       /tmp/ersilia-test-output-<model-id>.csv
   ```

2. **Validate output**: file exists, 3 rows, headers match run_columns.csv, values match run_output.csv, no invalid values (NaN/None/null).

3. **Consistency check**: run main.py a second time, compare outputs — must be identical for Fixed output consistency.

If end-to-end validation fails, fix the generated code and re-run until it passes.

### 5c. Run deep checks (scientific validation)

Using the verification venv from Phase 2, run the three deep validation sub-checks as described in `/test-model` Phase 3. Follow the same procedures from the eos-template-knowledge skill reference data.

**Distribution analysis**: Write the DIVERSE_50 set to a temp CSV, run main.py, compute per-column statistics (mean, std, min, max, coefficient of variation). Check completion rate >= 90%, non-constant outputs, all finite, runtime < 300s.

**Sanity checks**: Resolve Tags + Biomedical Area to sanity categories using TAG_TO_SANITY_KEY. For each matched category, run positive and negative reference compounds, verify directional separation matches run_columns.csv direction. For Representation models, check cosine similarity of similar_pair > dissimilar_pair.

**Paper reproduction** (only if `--paper` URL was provided): Fetch paper with WebFetch, extract reported metrics. Search source repo for test data. If found, run model on subset and compare metrics (20% tolerance = pass, 50% = warning). Paper reproduction never hard-fails.

Collect all deep check results into a `deep_checks` dictionary.

### 5d. Generate test report

Write a `test_report.json` file in the model directory root (`<output-dir>/<model-id>/test_report.json`). This file serves as evidence that the model was tested before publication.

The report must contain:

```json
{
  "model_id": "<model-id>",
  "test_date": "<ISO 8601 timestamp>",
  "test_tool": "ersilia-hub-plugin",
  "test_tool_version": "1.0.0",
  "source_repo": "<repo-url>",
  "paper_url": "<paper-url or null>",
  "python_version": "<python version used in test venv>",
  "verification_environment": {
    "packages": {
      "<package>": "<version>",
      ...
    }
  },
  "inspect_checks": {
    "file_existence": {
      "status": "passed",
      "files_checked": 7,
      "files_found": 7
    },
    "metadata_validation": {
      "status": "passed",
      "checks_passed": <N>,
      "checks_failed": 0,
      "details": {
        "<field>": {"status": "passed", "value": "<value>"},
        ...
      }
    },
    "column_consistency": {
      "status": "passed",
      "num_columns": <N>,
      "columns_match": true,
      "output_rows": 3,
      "input_rows": 3
    },
    "dependency_pinning": {
      "status": "passed",
      "python_version": "<version>",
      "num_packages": <N>,
      "all_pinned": true
    },
    "syntax_validation": {
      "status": "passed",
      "file": "model/framework/code/main.py"
    }
  },
  "shallow_checks": {
    "end_to_end_run": {
      "status": "passed",
      "exit_code": 0,
      "output_rows": 3,
      "output_columns": <N>,
      "values_match_expected": true,
      "no_invalid_values": true
    },
    "consistency": {
      "status": "passed",
      "method": "dual_run_comparison",
      "identical": true
    }
  },
  "deep_checks": {
    "distribution_analysis": {
      "status": "passed",
      "num_input_molecules": 50,
      "num_valid_outputs": <N>,
      "completion_rate": <float>,
      "runtime_seconds": <float>,
      "columns": {
        "<col_name>": {
          "mean": <float>,
          "std": <float>,
          "min": <float>,
          "max": <float>,
          "coefficient_of_variation": <float>,
          "is_constant": false,
          "all_finite": true,
          "status": "passed"
        }
      }
    },
    "sanity_checks": {
      "status": "passed|warning|skipped",
      "matched_categories": ["<category>"],
      "checks": [
        {
          "category": "<category>",
          "column": "<col_name>",
          "direction": "high|low",
          "positive_mean": <float>,
          "negative_mean": <float>,
          "separation_correct": true,
          "status": "passed"
        }
      ]
    },
    "paper_reproduction": {
      "status": "passed|warning|skipped",
      "paper_url": "<url or null>",
      "reported_metrics": [{"metric": "<name>", "value": <float>}],
      "reproduced_metrics": [{"metric": "<name>", "value": <float>, "deviation_pct": <float>}],
      "detail": "<explanation>"
    }
  },
  "verified_outputs": {
    "input_smiles": [
      "CC(=O)Oc1ccccc1C(=O)O",
      "CC(C)Cc1ccc(cc1)C(C)C(=O)O",
      "Cn1c(=O)c2c(ncn2C)n(c1=O)C"
    ],
    "output_values": [
      [<row 1 values>],
      [<row 2 values>],
      [<row 3 values>]
    ],
    "output_columns": ["<col1>", "<col2>", ...]
  },
  "overall": {
    "status": "passed",
    "total_checks": <N>,
    "passed": <N>,
    "warnings": 0,
    "failed": 0
  }
}
```

### 5e. Report summary

Display a summary table to the user showing all check results (inspect, shallow, and deep), plus confirmation that `test_report.json` was generated. List all generated files with their paths and sizes.

## Important Rules

- Generate FUNCTIONAL code. Never write TODO or placeholder comments instead of real logic.
- If you cannot determine how to call inference, explain what you found and ask the user rather than guessing.
- Always pin dependency versions. Use verified versions from the test environment's pip freeze.
- Only use vocabulary values from the eos-template-knowledge skill for metadata.yml fields.
- The output CSV must NOT contain the input SMILES column — only prediction columns.
- NEVER use placeholder values in run_output.csv — always use real outputs from verification.
- The running code is the source of truth. If your code analysis says one thing and execution says another, trust execution.
