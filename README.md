# ersilia-hub-plugin

A [Claude Code](https://claude.com/claude-code) plugin that automates the end-to-end incorporation of published ML models into the [Ersilia Model Hub](https://ersilia.io) -- from source code analysis to repository creation.

## Workflow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       /incorporate-model <url>                          │
│                                                                         │
│   Paper + Code ──> Analyze ──> Verify by ──> Assign ID ──> Generate    │
│                     source      Running      (eos7k2f)    eos-template  │
│                                  model                        |         │
│                                                         Test & Report   │
│                                                        test_report.json │
└───────────────────────────────────────────────────────────────┬─────────┘
                                                                |
                                              ┌─────────────────┘
                                              v
┌─────────────────────────────────────────────────────────────────────────┐
│                      /publish-model <dir>                               │
│                                                                         │
│   eos-template ──> Create repo ──> Open PR ──> Maintainer ──> CI Deploy │
│   + test_report   on ersilia-os   with test    reviews &     S3+Docker  │
│                                   evidence     merges                   │
└─────────────────────────────────────────────────────────────────────────┘
```

## What it does

This plugin provides three commands and a knowledge base that cover the full model incorporation workflow:

```
/incorporate-model <url>    ->  analyze, verify, assign ID, generate eos-template + test report
/test-model <model-dir>     ->  validate files and run checks locally (standalone)
/publish-model <model-dir>  ->  create repo under ersilia-os, open PR with test evidence
```

### `/incorporate-model`

Takes a published ML model (from GitHub or Zenodo) and generates a complete eos-template model directory. Runs through five phases:

1. **Clone & Analyze** -- clones the source repo, identifies the ML framework, dependencies, inference entry point, and input/output formats
2. **Verify by Running** -- installs the model in an isolated venv and runs inference on test molecules to confirm output dimensions, types, and values
3. **Present Verified Analysis** -- presents the analysis (confirmed by execution) and proposed metadata
4. **Generate Files** -- creates the full eos-template directory structure with functional code and real example outputs
5. **Test & Generate Report** -- runs inspect checks (file existence, metadata validation, column consistency, dependency pinning, syntax) and shallow checks (end-to-end run, output validation, consistency verification), then writes a `test_report.json` as evidence

### `/test-model`

Validates a locally generated model before publishing. Runs independently of the ersilia CLI:

1. **Inspect checks** -- file existence (7 mandatory files), metadata validation against ersilia vocabulary, column consistency between `run_columns.csv` and `run_output.csv`, dependency pinning in `install.yml`, Python syntax validation of `main.py`
2. **Shallow checks** -- end-to-end execution of `main.py` on example inputs, output validation (correct rows, columns, no invalid values), and consistency verification (dual-run comparison)

### `/publish-model`

Creates a new repository under `ersilia-os` from the `eos-template` and opens a pull request for maintainer review:

1. **Pre-publish validation** -- checks files, test report, model ID format, GitHub permissions, and whether the repo already exists
2. **Create repository** -- uses `gh repo create --template ersilia-os/eos-template`
3. **Populate on a branch** -- creates `incorporate/<model-id>` branch, copies generated files, updates the model identifier
4. **Open PR** -- commits, pushes the branch, and opens a PR with test report evidence (check results, verified outputs) in the body
5. **Post-publish** -- verifies PR was created, checks CI status, reports next steps for the reviewer

The PR includes a summary table of all test checks and the verified model outputs so maintainers can review without running tests locally. Supports `--dry-run` to preview without making changes.

### `eos-template-knowledge` skill

A knowledge base that Claude Code references automatically when working with eos-template repositories. Covers repository structure, the `main.py` contract, `install.yml` and `metadata.yml` formats, valid metadata vocabulary, and example file formats.

## Installation

Clone this repository into your Claude Code plugins directory:

```bash
git clone https://github.com/sergivalverde/ersilia-hub-plugin ~/.claude/plugins/ersilia-hub
```

Then enable the plugin by adding the following to your `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "ersilia-hub@ersilia-hub": true
  }
}
```

## Usage

### Full workflow example

```bash
# 1. Generate eos-template files from a published model (auto-assigns a unique model ID)
/incorporate-model https://github.com/author/model-repo --paper https://doi.org/10.1234/paper

# 2. Test locally before publishing (optional, /incorporate-model already tests)
/test-model ./eos7k2f

# 3. Preview what publishing will do
/publish-model ./eos7k2f --dry-run

# 4. Publish to ersilia-os (opens a PR for maintainer review)
/publish-model ./eos7k2f
```

### Command reference

#### `/incorporate-model`

| Argument | Required | Description |
|----------|----------|-------------|
| `<repo-url>` | Yes | GitHub or Zenodo URL of the source model |
| `--paper <url>` | No | URL of the associated publication |
| `--model-id <id>` | No | Ersilia model identifier (eosXXXX format). Auto-generated if not provided |
| `--output-dir <path>` | No | Where to create the model directory. Defaults to current working directory |

#### `/test-model`

| Argument | Required | Description |
|----------|----------|-------------|
| `<model-dir>` | Yes | Path to the local eos-template model directory |
| `--level <level>` | No | Test depth: `inspect`, `surface`, `shallow` (default), or `deep` |

#### `/publish-model`

| Argument | Required | Description |
|----------|----------|-------------|
| `<model-dir>` | Yes | Path to the local model directory to publish (ID read from metadata.yml) |
| `--org <org>` | No | GitHub organization. Defaults to `ersilia-os` |
| `--dry-run` | No | Preview actions without making changes |

### Test report (`test_report.json`)

Both `/incorporate-model` (Phase 5) and `/test-model` generate a `test_report.json` file in the model directory root. This file serves as evidence that the model was tested before publication and is committed alongside the model files.

The report contains:

```json
{
  "model_id": "eos0xxx",
  "test_date": "2026-02-20T07:46:01.349338+00:00",
  "test_tool": "ersilia-hub-plugin",
  "test_tool_version": "1.0.0",
  "source_repo": "https://github.com/author/model-repo",
  "paper_url": "https://doi.org/10.1234/paper",
  "python_version": "3.9.6",
  "verification_environment": {
    "packages": {
      "package-name": "version",
      "...": "..."
    }
  },
  "inspect_checks": {
    "file_existence":       { "status": "passed", "files_checked": 7, "files_found": 7 },
    "metadata_validation":  { "status": "passed", "checks_passed": 22, "checks_failed": 0, "details": { "...": "..." } },
    "column_consistency":   { "status": "passed", "num_columns": 39, "columns_match": true, "output_rows": 3, "input_rows": 3 },
    "dependency_pinning":   { "status": "passed", "python_version": "3.10", "num_packages": 4, "all_pinned": true },
    "syntax_validation":    { "status": "passed", "file": "model/framework/code/main.py" }
  },
  "shallow_checks": {
    "end_to_end_run": { "status": "passed", "exit_code": 0, "output_rows": 3, "output_columns": 39, "values_match_expected": true, "no_invalid_values": true },
    "consistency":    { "status": "passed", "method": "dual_run_comparison", "identical": true }
  },
  "verified_outputs": {
    "input_smiles": ["CC(=O)Oc1ccccc1C(=O)O", "..."],
    "output_values": [[0, 0, "..."], ["..."]],
    "output_columns": ["col1", "col2", "..."]
  },
  "overall": {
    "status": "passed",
    "total_checks": 7,
    "passed": 7,
    "failed": 0
  }
}
```

| Section | Description |
|---------|-------------|
| `model_id` | Ersilia model identifier |
| `test_date` | ISO 8601 timestamp of when the test was run |
| `verification_environment` | Exact package versions used during testing (from `pip freeze`) |
| `inspect_checks` | Static validation results: file existence, metadata against ersilia vocabulary, column consistency, dependency pinning, syntax |
| `shallow_checks` | Runtime validation: end-to-end `main.py` execution and dual-run consistency comparison |
| `verified_outputs` | Actual model outputs on the 3 test molecules (aspirin, ibuprofen, caffeine), used as ground truth |
| `overall` | Aggregate pass/fail status and check counts |

### Using the knowledge base

The eos-template-knowledge skill activates automatically when you ask Claude Code about incorporating models, the eos-template format, or when working inside an eosXXXX repository.

## Configuration

### Plugin settings

The plugin is configured via the `enabledPlugins` key in `~/.claude/settings.json`. Set it to `true` to enable or `false` to disable:

```json
{
  "enabledPlugins": {
    "ersilia-hub@ersilia-hub": true
  }
}
```

### Permissions

For a smoother experience, you can pre-allow common operations in your project's `.claude/settings.local.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(git clone:*)",
      "Bash(git add:*)",
      "Bash(git push:*)",
      "Bash(python3:*)",
      "Bash(gh repo create:*)",
      "Bash(gh repo view:*)",
      "Bash(gh run list:*)",
      "Bash(ersilia:*)"
    ]
  }
}
```

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI
- Git
- Python 3.9+
- [GitHub CLI](https://cli.github.com/) (`gh`) -- for `/publish-model`
- [Ersilia CLI](https://github.com/ersilia-os/ersilia) -- optional, the plugin runs tests independently

## License

GPL-3.0
