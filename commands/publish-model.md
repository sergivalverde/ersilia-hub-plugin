---
description: Create an ersilia-os repo from eos-template and push a locally generated model
argument-hint: <model-id> <model-dir> [--org <org>] [--dry-run]
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep, Task, AskUserQuestion, TodoWrite]
---

# Publish Model to Ersilia Model Hub

You are creating a new repository under the ersilia-os GitHub organization from the eos-template and populating it with a locally generated model.

## Parse Arguments

From the user's input, extract:
- `model-id` (required): The assigned Ersilia model identifier (eosXXXX format, e.g., `eos4abc`). Must NOT be `eos0xxx` or `eos0aaa` (these are reserved for testing/template).
- `model-dir` (required): Path to the local directory containing the generated eos-template files (e.g., `./eos0xxx`)
- `--org <org>` (optional): GitHub organization. Default: `ersilia-os`
- `--dry-run` (optional): If set, show what would happen without creating the repo or pushing

## Phase 1: Pre-publish Validation

### 1a. Verify the local model is ready

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

### 1b. Verify the model ID

1. Confirm `model-id` matches the pattern `eos[1-9][a-z0-9]{3}` (production IDs). Warn if it uses `eos0` prefix (test/placeholder range).
2. Check if the repo `<org>/<model-id>` already exists:
   ```bash
   gh repo view <org>/<model-id> --json name 2>/dev/null
   ```
   If it exists, warn the user and ask how to proceed:
   - Push to the existing repo (if it's empty/template-only)
   - Choose a different model ID
   - Abort

### 1c. Verify GitHub permissions

```bash
gh auth status
```

Confirm the authenticated user has permission to create repos under the target org. If not, inform the user and stop.

### 1d. Dry run summary

If `--dry-run` is set, present what would happen and stop:

```
## Dry Run Summary

- Create repo: <org>/<model-id> (public, from ersilia-os/eos-template)
- Clone to: /tmp/ersilia-publish-<model-id>
- Copy files from: <model-dir>
- Update metadata.yml Identifier: eos0xxx → <model-id>
- Commit and push to main
- CI workflows will trigger: test-model-pr.yml, upload-model.yml

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

## Phase 3: Populate Repository

### 3a. Copy generated files

Copy all files from `<model-dir>` into the cloned repo, replacing the template placeholders:

```bash
cp -r <model-dir>/model/framework/code/* <model-id>/model/framework/code/
cp -r <model-dir>/model/framework/examples/* <model-id>/model/framework/examples/
cp -r <model-dir>/model/framework/columns/* <model-id>/model/framework/columns/
cp <model-dir>/model/framework/run.sh <model-id>/model/framework/run.sh
cp <model-dir>/install.yml <model-id>/install.yml
cp <model-dir>/metadata.yml <model-id>/metadata.yml
```

If there are checkpoint files in `<model-dir>/model/checkpoints/`, copy those too:
```bash
cp -r <model-dir>/model/checkpoints/* <model-id>/model/checkpoints/
```

### 3b. Update the model identifier

Update `metadata.yml` to replace the placeholder identifier with the real one:
- Change `Identifier: eos0xxx` (or whatever placeholder) to `Identifier: <model-id>`
- Change `Slug:` if it contains placeholder text

### 3c. Check for large files

Check if any files exceed GitHub's 100MB limit:
```bash
find <model-id> -type f -size +50M
```

If large files are found, warn the user about Git LFS requirements. Files over 100MB will fail to push. Suggest:
- Setting up Git LFS: `git lfs track "model/checkpoints/*.pt"`
- Or hosting checkpoints externally and downloading in `install.yml`

## Phase 4: Commit and Push

### 4a. Stage and commit

```bash
cd <model-id>
git add -A
git commit -m "Add <model-name> model

Automated incorporation via ersilia-hub-plugin.
- Source: <source-code-url from metadata.yml>
- Paper: <publication-url from metadata.yml>"
```

### 4b. Push to remote

```bash
git push origin main
```

This will trigger the CI workflows defined in the template:
- `test-model-pr.yml` (if pushing via PR)
- `upload-model.yml` (on push to main — tests, uploads to S3, builds Docker image)

## Phase 5: Post-publish Verification

### 5a. Verify push succeeded

```bash
gh repo view <org>/<model-id> --json name,url,defaultBranchRef
```

### 5b. Check CI status

Wait briefly then check if workflows started:

```bash
sleep 10
gh run list --repo <org>/<model-id> --limit 3
```

### 5c. Report summary

```
## Published: <org>/<model-id>

- Repository: https://github.com/<org>/<model-id>
- Files pushed: [list of files]
- CI status: [running/pending/completed]

### Next Steps
1. Monitor CI at: https://github.com/<org>/<model-id>/actions
2. Once CI passes, the model will be:
   - Uploaded to S3
   - Built as Docker image: ersiliaos/<model-id>
3. A test issue will be auto-created for manual validation
4. Run: ersilia -v test <model-id> --shallow --from_dockerhub
```

## Important Rules

- NEVER publish with a placeholder ID (`eos0xxx`, `eos0aaa`). Always require a real model ID.
- Always check if the repo already exists before creating.
- Always update the Identifier in metadata.yml to match the repo name.
- If checkpoint files are large (>50MB), warn about Git LFS before pushing.
- The `--dry-run` flag should be used first to preview what will happen.
- Do NOT modify the `.github/workflows/` directory — the CI comes pre-configured from the template.
- Do NOT modify `README.md` — it is auto-generated by Ersilia infrastructure.
