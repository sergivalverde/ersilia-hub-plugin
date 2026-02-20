# ersilia-hub-plugin

A [Claude Code](https://claude.com/claude-code) plugin that automates the incorporation of published ML models into the [Ersilia Model Hub](https://ersilia.io) by wrapping them in the standardized eos-template format.

## What it does

This plugin provides two components:

### `/incorporate-model` command

An interactive command that takes a published ML model (from GitHub or Zenodo) and generates a complete eos-template model repository, including:

- A functional `main.py` wrapper (CSV-in / CSV-out)
- `run.sh` entry point
- `install.yml` with pinned dependencies
- `metadata.yml` with validated vocabulary
- Example input/output files
- Output column definitions

The command runs through four phases:

1. **Clone & Analyze** -- clones the source repo, identifies the ML framework, dependencies, inference entry point, and input/output formats
2. **Plan & Confirm** -- presents the analysis and proposed metadata for your approval before generating anything
3. **Generate Files** -- creates the full eos-template directory structure with functional (not placeholder) code
4. **Validate** -- verifies all required files exist and are syntactically valid

### `eos-template-knowledge` skill

A knowledge base that Claude Code can reference whenever you're working with eos-template repositories. It covers:

- Repository structure and file conventions
- The `main.py` contract and common inference patterns
- `install.yml` and `metadata.yml` formats
- Valid metadata vocabulary (tasks, tags, biomedical areas, organisms, licenses, etc.)
- Column definitions and example file formats

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

### Incorporating a model

In Claude Code, run:

```
/incorporate-model https://github.com/author/model-repo
```

With optional flags:

```
/incorporate-model https://github.com/author/model-repo --paper https://doi.org/10.1234/paper --model-id eos4e40 --output-dir ./models
```

| Argument | Required | Description |
|----------|----------|-------------|
| `<repo-url>` | Yes | GitHub or Zenodo URL of the source model |
| `--paper <url>` | No | URL of the associated publication |
| `--model-id <id>` | No | Ersilia model identifier (eosXXXX format). Defaults to `eos0xxx` |
| `--output-dir <path>` | No | Where to create the model directory. Defaults to current working directory |

### Using the knowledge base

The eos-template-knowledge skill activates automatically when you ask Claude Code about incorporating models, the eos-template format, or when working inside an eosXXXX repository. You can also reference it directly by asking about any aspect of the eos-template structure.

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

The `/incorporate-model` command uses the following tools: `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `WebFetch`, `Task`, `AskUserQuestion`, and `TodoWrite`. Claude Code will prompt you for approval on any tool calls that aren't already allowed in your permission settings.

For a smoother experience, you can pre-allow common operations in your project's `.claude/settings.local.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(git clone:*)",
      "Bash(python -c:*)",
      "Bash(curl:*)"
    ]
  }
}
```

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI
- Git (for cloning source repositories)
- Python (for validating generated files)

## License

GPL-3.0
