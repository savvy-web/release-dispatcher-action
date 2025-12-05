# Release Dispatcher Action

Unified release workflow dispatcher for Savvy Web repositories. Provides centralized release management with npm provenance support.

## Features

- **Centralized Workflows**: All release logic in one place
- **npm Provenance**: Publish with supply chain attestation
- **Phase Detection**: Automatically detects branch-management, validation, or publishing phase
- **PR Validation**: Automated PR title, commit, lint, and test checks
- **Minimal Per-Repo Config**: Just ~20 lines per repository

## Architecture

```
release-dispatcher-action/
├── .github/workflows/
│   ├── release.yml           ← Main dispatcher (entry point)
│   ├── release-branch.yml    ← Branch management phase
│   ├── release-publish.yml   ← Publishing phase (npm provenance)
│   └── release-validate.yml  ← PR validation phase
```

## Usage

Add this to your repository's `.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    branches: [main]
  pull_request:
    branches: [main, changeset-release/main]
    types: [opened, synchronize, reopened, edited, closed]
  workflow_dispatch:
    inputs:
      dry_run:
        description: Run in dry-run mode
        type: boolean
        default: true

permissions:
  contents: write
  pull-requests: write
  id-token: write
  packages: write
  attestations: write
  checks: write

jobs:
  release:
    uses: savvy-web/release-dispatcher-action/.github/workflows/release.yml@main
    with:
      dry-run: ${{ inputs.dry_run || false }}
    secrets: inherit
```

## Required Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `APP_ID` | Yes | GitHub App ID |
| `APP_PRIVATE_KEY` | Yes | GitHub App private key |
| `ANTHROPIC_API_KEY` | No | For Claude-powered PR descriptions |
| `CLAUDE_REVIEW_PAT` | No | For Claude code review |
| `SAVVYWEB_NPM_AUTH_TOKEN` | No | For custom registry publishing |

## Workflow Phases

| Phase | Trigger | Action |
|-------|---------|--------|
| `branch-management` | Push to main (non-release) | Creates/updates release branch |
| `validation` | PR opened/updated | Runs PR checks and pre-publish validation |
| `publishing` | Release PR merged | Publishes packages with provenance |
| `close-issues` | Release PR merged | Closes linked issues |

## Dependencies

- [workflow-control-action](https://github.com/savvy-web/workflow-control-action) - Phase detection
- [workflow-runtime-action](https://github.com/savvy-web/workflow-runtime-action) - Runtime setup
- [workflow-release-action](https://github.com/savvy-web/workflow-release-action) - Release logic

## License

MIT
