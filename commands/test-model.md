---
description: Run local tests on an eos-template model before publishing
argument-hint: <model-dir> [--level inspect|surface|shallow|deep]
allowed-tools: [Bash, Read, Glob, Grep, Task, TodoWrite, AskUserQuestion]
---

# Test Model Locally

You are testing a locally generated eos-template model to validate it before publishing to the Ersilia Model Hub.

## Parse Arguments

From the user's input, extract:
- `model-dir` (required): Path to the local eos-template model directory (e.g., `./eos0xxx`)
- `--level <level>` (optional): Test depth. One of `inspect`, `surface`, `shallow`, `deep`. Default: `shallow`

## Phase 1: Pre-flight Checks

Before running ersilia test, perform quick local validation:

### 1a. Verify required files exist

Check that all 7 mandatory files are present:
```
model/framework/code/main.py
model/framework/run.sh
model/framework/examples/run_input.csv
model/framework/examples/run_output.csv
model/framework/columns/run_columns.csv
install.yml
metadata.yml
```

If any are missing, report the missing files and stop.

### 1b. Validate metadata.yml

Read `metadata.yml` and check:
1. **Identifier** matches the directory name (e.g., `eos0xxx`)
2. **Title** is between 10 and 300 characters
3. **Description** is at least 200 characters and differs from Title
4. **Interpretation** is between 10 and 300 characters
5. **Task** is one of: Representation, Annotation, Sampling
6. **Subtask** is one of: Featurization, Projection, Property calculation or prediction, Activity prediction, Similarity search, Generation
7. **Input** is one of: Compound, Protein, Text
8. **Output** is one of: Compound, Score, Value, Text
9. **Output Consistency** is one of: Fixed, Variable
10. **Publication** is a valid URL
11. **Source Code** is a valid URL
12. **License** is a valid SPDX identifier from the eos-template-knowledge skill vocabulary

Report any validation errors and ask the user if they want to fix them before continuing.

### 1c. Validate column consistency

1. Read `model/framework/columns/run_columns.csv` and extract column names
2. Read `model/framework/examples/run_output.csv` and extract header columns
3. Verify they match exactly (same names, same order)
4. Verify `run_output.csv` has exactly 3 data rows (matching the 3 input SMILES)
5. Verify no `smiles` or `input` column appears in the output

### 1d. Validate main.py syntax

```bash
python3 -c "import ast; ast.parse(open('<model-dir>/model/framework/code/main.py').read())"
```

Report results for all pre-flight checks. If there are failures, ask the user whether to continue or stop to fix.

## Phase 2: Run ersilia test

### 2a. Check ersilia is installed

```bash
ersilia --version
```

If ersilia is not installed, inform the user:
```
ersilia CLI is not installed. Install it with:
  pip install ersilia

Or for testing capabilities:
  pip install ersilia[test]

Pre-flight checks passed, so the model structure is valid. Install ersilia to run the full test suite.
```

Stop here if ersilia is not available.

### 2b. Run the test

Run ersilia test at the requested level using `--from_dir`:

```bash
ersilia -v test <model-id> --<level> --from_dir <model-dir> --report_path /tmp/ersilia-test-reports
```

Where `<model-id>` is extracted from the directory name or `metadata.yml` Identifier field.

### 2c. Parse and report results

1. Read the JSON report at `/tmp/ersilia-test-reports/<model-id>-test.json`
2. Present results in a clear table:

```
## Test Results: <model-id>

| Check | Result |
|-------|--------|
| Metadata validation | PASS/FAIL |
| File structure | PASS/FAIL |
| Model fetch | PASS/FAIL |
| Model run | PASS/FAIL |
| Output consistency | PASS/FAIL |
| ... | ... |

Overall: PASS / FAIL (X of Y checks passed)
```

3. For any failed checks, explain what went wrong and suggest fixes.

## Phase 3: Cleanup

After reporting results, clean up the test artifacts:

```bash
ersilia delete <model-id> 2>/dev/null
```

## Important Rules

- Always run pre-flight checks first — they catch common issues faster than the full ersilia test suite.
- If pre-flight checks fail, the user should fix issues before running the full test.
- Never modify the model files during testing. This command is read-only.
- If ersilia is not installed, the pre-flight checks alone still provide value — report them and stop gracefully.
