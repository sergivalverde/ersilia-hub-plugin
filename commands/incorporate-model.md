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

## Phase 2: Plan and Confirm

Present your analysis to the user:

```
## Model Analysis

**What it does**: [one paragraph]
**ML Framework**: [framework name]
**Input**: [format description]
**Output**: [format description]
**Dependencies**: [key packages with versions]
**Checkpoint files**: [list with sizes]
**Inference entry point**: [file and function/method]

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

Then ask the user: "Does this look correct? Should I proceed with generating the eos-template files, or would you like to adjust anything?"

Wait for confirmation before proceeding.

## Phase 3: Generate Files

After user confirmation, generate all files:

### 3a. Create directory structure
```
mkdir -p <output-dir>/<model-id>/model/checkpoints
mkdir -p <output-dir>/<model-id>/model/framework/code
mkdir -p <output-dir>/<model-id>/model/framework/examples
mkdir -p <output-dir>/<model-id>/model/framework/columns
```

### 3b. Write model/framework/code/main.py

This MUST be functional code, not a placeholder. Follow the main.py contract from the eos-template-knowledge skill:
- `sys.argv[1]` = input CSV, `sys.argv[2]` = output CSV
- Read SMILES from input CSV (skip header, take first column)
- Call the actual model inference
- Assert input length equals output length
- Write output CSV with appropriate headers (matching run_columns.csv)

Copy any source code files from the cloned repo that main.py needs into `model/framework/code/`. Adjust import paths accordingly.

### 3c. Copy checkpoint files
Copy pre-trained weights from the source repo into `model/checkpoints/`.

### 3d. Write model/framework/run.sh
```
python $1/code/main.py $2 $3
```

### 3e. Write install.yml
Pin all dependency versions. Use the source repo's dependency files as reference. Prefer Python 3.10 unless the model requires a specific version.

### 3f. Write metadata.yml
Use ONLY values from the valid vocabulary lists in the eos-template-knowledge skill. Set Status to "In progress". Set Source Type to "External" for third-party models.

### 3g. Write example files
- `model/framework/examples/run_input.csv`: 3 SMILES with header `smiles`. Use aspirin (`CC(=O)Oc1ccccc1C(=O)O`), ibuprofen (`CC(C)Cc1ccc(cc1)C(C)C(=O)O`), caffeine (`Cn1c(=O)c2c(ncn2C)n(c1=O)C`) as safe defaults.
- `model/framework/examples/run_output.csv`: placeholder values matching the expected output format
- `model/framework/columns/run_columns.csv`: column metadata with name, type, direction, description

## Phase 4: Validate

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

4. Report a summary listing all generated files with their paths and sizes.

## Important Rules

- Generate FUNCTIONAL code. Never write TODO or placeholder comments instead of real logic.
- If you cannot determine how to call inference, explain what you found and ask the user rather than guessing.
- Always pin dependency versions. If unpinned in source, check the repo's last commit date and use versions from that era.
- Only use vocabulary values from the eos-template-knowledge skill for metadata.yml fields.
- The output CSV must NOT contain the input SMILES column — only prediction columns.
