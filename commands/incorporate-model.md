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
- `--model-id <id>` (optional): Ersilia model identifier (eosXXXX format). Default: `eos0xxx`
- `--output-dir <path>` (optional): Where to create the model directory. Default: current working directory

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

## Phase 3: Present Verified Analysis

Present your **verified** analysis to the user, clearly marking what was confirmed by execution:

```
## Verified Model Analysis

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

Then ask the user: "This analysis has been verified by running the model. Should I proceed with generating the eos-template files, or would you like to adjust anything?"

Wait for confirmation before proceeding.

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

## Phase 5: Validate

### 5a. Static validation

1. Verify all 7 required files exist:
   - model/framework/code/main.py
   - model/framework/run.sh
   - model/framework/examples/run_input.csv
   - model/framework/examples/run_output.csv
   - model/framework/columns/run_columns.csv
   - install.yml
   - metadata.yml

2. Verify main.py is syntactically valid:
   ```
   python -c "import ast; ast.parse(open('<path>/model/framework/code/main.py').read())"
   ```

3. Verify metadata.yml is valid YAML:
   ```
   python -c "import yaml; yaml.safe_load(open('<path>/metadata.yml'))"
   ```

### 5b. End-to-end validation

Run the generated main.py against the example input to confirm it produces correct output:

```bash
/tmp/ersilia-verify-<model-id>/bin/python3 <path>/model/framework/code/main.py \
    <path>/model/framework/examples/run_input.csv \
    /tmp/ersilia-test-output-<model-id>.csv
```

Then verify:
1. The output file was created
2. It has the correct number of rows (3, matching input)
3. It has the correct column headers (matching run_columns.csv)
4. The values match the verified outputs from Phase 2d
5. No input/SMILES column appears in the output

If end-to-end validation fails, fix the generated code and re-run until it passes.

### 5c. Report summary

Report a summary listing all generated files with their paths and sizes, plus confirmation that end-to-end validation passed.

## Important Rules

- Generate FUNCTIONAL code. Never write TODO or placeholder comments instead of real logic.
- If you cannot determine how to call inference, explain what you found and ask the user rather than guessing.
- Always pin dependency versions. Use verified versions from the test environment's pip freeze.
- Only use vocabulary values from the eos-template-knowledge skill for metadata.yml fields.
- The output CSV must NOT contain the input SMILES column — only prediction columns.
- NEVER use placeholder values in run_output.csv — always use real outputs from verification.
- The running code is the source of truth. If your code analysis says one thing and execution says another, trust execution.
