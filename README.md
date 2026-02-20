# ersilia-hub-plugin

A [Claude Code](https://claude.com/claude-code) plugin that automates the end-to-end incorporation of published ML models into the [Ersilia Model Hub](https://ersilia.io) -- from source code analysis to repository creation.

## What it does

This plugin provides three commands and a knowledge base that cover the full model incorporation workflow:

```
/incorporate-model <url>    →  analyze source, verify by running, generate eos-template files
/test-model <model-dir>     →  validate files and run ersilia test locally
/publish-model <id> <dir>   →  create repo under ersilia-os, push, trigger CI
```

### `/incorporate-model`

Takes a published ML model (from GitHub or Zenodo) and generates a complete eos-template model directory. Runs through five phases:

1. **Clone & Analyze** -- clones the source repo, identifies the ML framework, dependencies, inference entry point, and input/output formats
2. **Verify by Running** -- installs the model in an isolated venv and runs inference on test molecules to confirm output dimensions, types, and values
3. **Present Verified Analysis** -- presents the analysis (confirmed by execution) and proposed metadata
4. **Generate Files** -- creates the full eos-template directory structure with functional code and real example outputs
5. **Validate** -- static checks plus end-to-end validation of the generated `main.py`

### `/test-model`

Validates a locally generated model before publishing. Two levels of testing:

1. **Pre-flight checks** (no ersilia required) -- verifies file existence, metadata vocabulary, column consistency, and Python syntax
2. **ersilia test** (requires ersilia CLI) -- runs the full test suite (`--inspect`, `--surface`, `--shallow`, or `--deep`) using `--from_dir`

### `/publish-model`

Creates a new repository under `ersilia-os` from the `eos-template` and pushes the generated files:

1. **Pre-publish validation** -- checks files, model ID format, GitHub permissions, and whether the repo already exists
2. **Create repository** -- uses `gh repo create --template ersilia-os/eos-template`
3. **Populate** -- copies generated files, updates the model identifier, checks for large files
4. **Push** -- commits and pushes to `main`, triggering CI workflows (S3 upload, Docker build)
5. **Post-publish** -- verifies push, checks CI status, reports next steps

Supports `--dry-run` to preview without making changes.

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
# 1. Generate eos-template files from a published model
/incorporate-model https://github.com/author/model-repo --paper https://doi.org/10.1234/paper

# 2. Test locally before publishing
/test-model ./eos0xxx

# 3. Preview what publishing will do
/publish-model eos4abc ./eos0xxx --dry-run

# 4. Publish to ersilia-os
/publish-model eos4abc ./eos0xxx
```

### Command reference

#### `/incorporate-model`

| Argument | Required | Description |
|----------|----------|-------------|
| `<repo-url>` | Yes | GitHub or Zenodo URL of the source model |
| `--paper <url>` | No | URL of the associated publication |
| `--model-id <id>` | No | Ersilia model identifier (eosXXXX format). Defaults to `eos0xxx` |
| `--output-dir <path>` | No | Where to create the model directory. Defaults to current working directory |

#### `/test-model`

| Argument | Required | Description |
|----------|----------|-------------|
| `<model-dir>` | Yes | Path to the local eos-template model directory |
| `--level <level>` | No | Test depth: `inspect`, `surface`, `shallow` (default), or `deep` |

#### `/publish-model`

| Argument | Required | Description |
|----------|----------|-------------|
| `<model-id>` | Yes | Assigned Ersilia model ID (eosXXXX format) |
| `<model-dir>` | Yes | Path to the local model directory to publish |
| `--org <org>` | No | GitHub organization. Defaults to `ersilia-os` |
| `--dry-run` | No | Preview actions without making changes |

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
- [Ersilia CLI](https://github.com/ersilia-os/ersilia) -- for `/test-model` full test suite (pre-flight checks work without it)

## License

GPL-3.0
