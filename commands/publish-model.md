---
description: Create an ersilia-os repo from eos-template and open a PR with the model files
argument-hint: <model-dir> [--org <org>] [--dry-run]
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep, Task, AskUserQuestion, TodoWrite]
---

# Publish Model to Ersilia Model Hub

You are creating a new repository under the ersilia-os GitHub organization from the eos-template and opening a pull request with the model files for maintainer review.

## Parse Arguments

From the user's input, extract:
- `model-dir` (required): Path to the local directory containing the generated eos-template files (e.g., `./eos7k2f`)
- `--org <org>` (optional): GitHub organization. Default: `ersilia-os`
- `--dry-run` (optional): If set, show what would happen without creating the repo or pushing

The model ID is read from the `Identifier` field in `<model-dir>/metadata.yml`. It was assigned during `/incorporate-model`.

## Phase 1: Pre-publish Validation

### 1a. Read the model ID from metadata

```bash
python3 -c "import yaml; print(yaml.safe_load(open('<model-dir>/metadata.yml'))['Identifier'])"
```

This is the `model-id` used throughout the rest of the process. If the Identifier is `eos0xxx` or `eos0aaa`, stop and tell the user to re-run `/incorporate-model` to get a real model ID assigned.

### 1b. Verify the local model is ready

Check that all 7 mandatory files exist in `<model-dir>`:
```
model/framework/code/main.py
model/framework/run.sh
model/framework/examples/run_input.csv
model/framework/examples/run_output.csv
model/framework/columns/run_columns.csv
install.yml
metadata.yml
```

If any are missing, stop and tell the user to run `/incorporate-model` first.

### 1c. Verify test report exists

Check that `<model-dir>/test_report.json` exists and that all checks passed:

```bash
python3 -c "
import json
with open('<model-dir>/test_report.json') as f:
    report = json.load(f)
print(f'Status: {report[\"overall\"][\"status\"]}')
print(f'Passed: {report[\"overall\"][\"passed\"]}/{report[\"overall\"][\"total_checks\"]}')
"
```

If the test report is missing or has failures, stop and tell the user to run `/test-model` or `/incorporate-model` first.

### 1d. Verify the model ID on GitHub

1. Confirm `model-id` matches the pattern `eos[1-9][a-z0-9]{3}` (production IDs).
2. Check if the repo `<org>/<model-id>` already exists:
   ```bash
   gh repo view <org>/<model-id> --json name 2>/dev/null
   ```
   If it exists, warn the user and ask how to proceed:
   - Push a branch and open a PR on the existing repo
   - Choose a different model ID
   - Abort

### 1e. Verify GitHub permissions

```bash
gh auth status
```

Confirm the authenticated user has permission to create repos under the target org. If not, inform the user and stop.

### 1f. Dry run summary

If `--dry-run` is set, present what would happen and stop:

```
## Dry Run Summary

- Create repo: <org>/<model-id> (public, from ersilia-os/eos-template)
- Clone to: /tmp/ersilia-publish-<model-id>
- Copy files from: <model-dir> (ID already set in metadata.yml)
- Create branch: incorporate/<model-id>
- Generate model-specific README.md
- Open PR with test report summary for maintainer review
- CI workflow test-model-pr.yml will run on the PR

No changes were made.
```

## Phase 2: Create Repository

### 2a. Create from template

```bash
gh repo create <org>/<model-id> \
  --template ersilia-os/eos-template \
  --public \
  --clone \
  --description "<Title from metadata.yml>"
```

This clones the new repo to the current directory as `./<model-id>`.

### 2b. Wait for repo initialization

GitHub takes a moment to initialize template repos. Wait and verify:

```bash
sleep 5
cd <model-id> && git log --oneline -1
```

If the repo is empty, wait a few more seconds and retry.

## Phase 3: Populate on a Branch

### 3a. Create a feature branch

```bash
cd <model-id>
git checkout -b incorporate/<model-id>
```

### 3b. Copy generated files

Copy all files from `<model-dir>` into the cloned repo, replacing the template placeholders:

```bash
cp -r <model-dir>/model/framework/code/* <model-id>/model/framework/code/
cp -r <model-dir>/model/framework/examples/* <model-id>/model/framework/examples/
cp -r <model-dir>/model/framework/columns/* <model-id>/model/framework/columns/
cp <model-dir>/model/framework/run.sh <model-id>/model/framework/run.sh
cp <model-dir>/install.yml <model-id>/install.yml
cp <model-dir>/metadata.yml <model-id>/metadata.yml
cp <model-dir>/test_report.json <model-id>/test_report.json
```

If `<model-dir>/deep_validation.ipynb` exists, copy it too:
```bash
cp <model-dir>/deep_validation.ipynb <model-id>/deep_validation.ipynb
```

If there are checkpoint files in `<model-dir>/model/checkpoints/`, copy those too:
```bash
cp -r <model-dir>/model/checkpoints/* <model-id>/model/checkpoints/
```

### 3c. Check for large files

Check if any files exceed GitHub's 100MB limit:
```bash
find <model-id> -type f -size +50M
```

If large files are found, warn the user about Git LFS requirements. Files over 100MB will fail to push. Suggest:
- Setting up Git LFS: `git lfs track "model/checkpoints/*.pt"`
- Or hosting checkpoints externally and downloading in `install.yml`

## Phase 4: Generate README

Replace the template `README.md` with a model-specific README built from `metadata.yml`, `test_report.json`, and `install.yml`. Write the generated content to `<model-id>/README.md`, overwriting the template default.

### README template

````markdown
# <Title from metadata.yml>

**Ersilia Model ID:** `<model-id>`

| | |
|---|---|
| **Task** | <Task from metadata.yml> |
| **Input** | <Input type and dimension> |
| **Output** | <Output type and dimension> |
| **Framework** | <Framework name and version from install.yml> |
| **License** | <License from metadata.yml> |

## Description

<Description from metadata.yml — full text>

## Interpretation

<Interpretation from metadata.yml — full text>

## Source

- **Publication:** [<Author et al., Journal Year>](<Publication URL>)
- **Source Code:** <Source Code URL>

## Deep Validation

| Check | Status | Details |
|-------|--------|---------|
<For each check in test_report.json, one row with check name, PASS/WARNING/FAIL, and key metrics>

**Overall: <STATUS> (<N>/<total>)**

### Highlights

<Write 2-4 bullet points summarizing the key validation findings:>
<- Distribution analysis: N molecules, runtime, completion rate, key stats>
<- Sanity check: what was compared, separation magnitude and direction>
<- Paper reproduction: what was validated, how results compare>
<- Any warnings or notable findings>

See [`deep_validation.ipynb`](deep_validation.ipynb) for full analysis with visualizations.

## Usage

```python
# Input CSV format
# <show the expected header and 2 example rows from run_input.csv>

python model/framework/code/main.py input.csv output.csv
```

## Dependencies

```yaml
<Contents of install.yml>
```
````

### Field extraction rules

- **Task**: From `metadata.yml` Task field. If Subtask exists, show as "Task / Subtask"
- **Input**: From Input + Input Dimension + Input Shape. Examples: "Single SMILES", "Compound pair (two SMILES)"
- **Output**: From Output Type + Output Dimension + Interpretation. Examples: "39-dimensional integer count vector", "4 log10-transformed ADME scores", "Score between 0 and 1"
- **Framework**: Infer from `install.yml` — the primary ML package and its version (e.g., "biosynfoni v1.0.0", "Chemprop v2.1.0", "PyTorch MLP on ECFP4 fingerprints")
- **Publication link text**: Use "Author et al., Journal Year" format. Extract first author surname and year from the URL or metadata
- **Deep Validation table**: Iterate over `test_report.json` → `deep_checks` keys. For each sub-check, extract status and the most important metric (e.g., completion rate, separation magnitude, similarity ratio)
- **Highlights**: Write in plain scientific language. Include specific numbers. Reference the paper's claim and how the model's behavior validates it

## Phase 5: Commit, Push, and Open PR

### 5a. Stage and commit

```bash
cd <model-id>
git add -A
git commit -m "Add <model-name> model

Automated incorporation via ersilia-hub-plugin.
- Source: <source-code-url from metadata.yml>
- Paper: <publication-url from metadata.yml>"
```

### 5b. Push the branch

```bash
git push -u origin incorporate/<model-id>
```

### 5c. Build the PR body from the test report

Read the `test_report.json` and build a PR description that includes:
- Model summary (from metadata.yml: title, description, task, input/output)
- Test evidence (from test_report.json: all check results, verified outputs)
- Source information (paper URL, source code URL)

### 5d. Open the pull request

```bash
gh pr create \
  --repo <org>/<model-id> \
  --base main \
  --head incorporate/<model-id> \
  --title "Add <model-name> model" \
  --body "$(cat <<'EOF'
## Model Summary

- **Title**: <title from metadata.yml>
- **Task**: <task> / <subtask>
- **Input**: <input type> → **Output**: <output type> (<output dimension> values)
- **Source**: [Paper](<publication-url>) | [Code](<source-code-url>)
- **License**: <license>

## Description

<description from metadata.yml>

## Pre-publication Test Report

All checks were run locally using `ersilia-hub-plugin` before submission.

### Inspect Checks

| Check | Status | Details |
|-------|--------|---------|
| File existence | ✅ passed | <files_found>/<files_checked> files |
| Metadata validation | ✅ passed | <checks_passed> fields validated |
| Column consistency | ✅ passed | <num_columns> columns, headers match |
| Dependency pinning | ✅ passed | Python <py_version>, <num_packages> packages pinned |
| Syntax validation | ✅ passed | main.py parses correctly |

### Shallow Checks

| Check | Status | Details |
|-------|--------|---------|
| End-to-end run | ✅ passed | <output_rows> rows, <output_columns> columns |
| Consistency | ✅ passed | Dual-run comparison identical |

### Deep Validation (if deep_checks present in test_report.json)

| Check | Status | Details |
|-------|--------|---------|
| Distribution analysis | ✅ passed | <num_valid_outputs>/<num_input_molecules> valid, CV=<cv> |
| Sanity (<category>) | ✅ passed | <positive_mean> vs <negative_mean> (correct separation) |
| Paper reproduction | ⚠️ skipped | <detail> |

#### Scientific Validation Analysis

Write a 2-3 paragraph narrative section here that explains:

1. **What was tested and why**: Describe the deep validation approach used for this specific model — what compounds were run, what properties were measured, and what the expected behavior is based on the paper's claims.

2. **Results obtained**: Report the key quantitative results — distribution statistics (completion rate, number of varying dimensions, runtime), sanity check metrics (similarity scores, separation ratios, directional correctness), and paper reproduction metrics (within/between-class similarity, metric deviation from paper). Include specific numbers.

3. **Correlation with the original paper**: Explain how the results confirm or deviate from the paper's claims. Reference the paper's key finding and show how the model's behavior on the test set aligns with it. If there are warnings or deviations, explain why they are acceptable (e.g., sparse fingerprints naturally have constant dimensions, wrapping may differ from the paper's exact pipeline).

This section should be model-specific and written in plain scientific language so a reviewer can understand the validation without opening the notebook.

### Verified Outputs (3 test molecules)

| SMILES | Output (first 5 values...) |
|--------|---------------------------|
| <smiles1> | <first 5 values> |
| <smiles2> | <first 5 values> |
| <smiles3> | <first 5 values> |

Full test report: [`test_report.json`](test_report.json)
Deep validation notebook with interactive plots: [`deep_validation.ipynb`](deep_validation.ipynb)

---
🤖 Automated by [ersilia-hub-plugin](https://github.com/sergivalverde/ersilia-hub-plugin)
EOF
)"
```

Replace all `<placeholders>` with actual values from `metadata.yml` and `test_report.json`. Use ✅ for passed checks and ❌ for failed ones.

## Phase 6: Post-publish Verification

### 6a. Verify PR was created

```bash
gh pr view --repo <org>/<model-id> --json number,url,state,title
```

### 6b. Check CI status

Wait briefly then check if the PR triggered CI workflows:

```bash
sleep 10
gh pr checks --repo <org>/<model-id>
```

### 6c. Report summary

```
## PR Opened: <org>/<model-id>#<pr-number>

- Pull Request: <pr-url>
- Branch: incorporate/<model-id> → main
- Files included: [list of files]
- Test report: ✅ all checks passed
- CI status: [running/pending/completed]

### Next Steps
1. A maintainer reviews the PR at: <pr-url>
2. CI runs `test-model-pr.yml` automatically on the PR
3. Once approved and merged:
   - `upload-model.yml` triggers on main (S3 upload, Docker build)
   - Docker image: ersiliaos/<model-id>
4. Post-merge validation: ersilia -v test <model-id> --shallow --from_dockerhub
```

## Important Rules

- NEVER publish with a placeholder ID (`eos0xxx`, `eos0aaa`). Always require a real model ID.
- Always check if the repo already exists before creating.
- Always include the test_report.json in the PR — it is the evidence of pre-publication testing.
- Always generate a model-specific README.md in Phase 4 — the template default must be replaced before the PR is opened.
- If checkpoint files are large (>50MB), warn about Git LFS before pushing.
- The `--dry-run` flag should be used first to preview what will happen.
- Do NOT modify the `.github/workflows/` directory — the CI comes pre-configured from the template.
- The PR body MUST include test results from test_report.json so reviewers can see evidence without running tests.
