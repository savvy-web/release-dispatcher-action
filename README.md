# Release Dispatcher Action

Unified release workflow dispatcher that provides centralized release management with npm provenance support. Deploy once, use across all your repositories.

## Why Use This?

| Problem | Solution |
|---------|----------|
| Duplicate workflow files across repos | Single source of truth |
| Inconsistent release processes | Standardized flow for all repos |
| Complex per-repo configuration | ~25 lines per repo |
| Scattered npm provenance | Centralized trusted publishing |
| Maintenance burden | Update once, all repos benefit |

## Architecture

```
Your Organization
├── release-dispatcher-action/        ← Central workflows (deploy once)
│   └── .github/workflows/
│       ├── release.yml               ← Main dispatcher
│       ├── release-branch.yml        ← Branch management
│       ├── release-publish.yml       ← Publishing (npm provenance)
│       └── release-validate.yml      ← PR validation
│
├── repo-a/                           ← Your repos (~25 lines each)
│   └── .github/workflows/
│       └── release.yml               ← Calls dispatcher
│
├── repo-b/
│   └── .github/workflows/
│       └── release.yml
│
└── repo-c/
    └── .github/workflows/
        └── release.yml
```

## Quick Start

### 1. Fork or Copy This Repository

Fork `savvy-web/release-dispatcher-action` to your organization, or copy the workflow files to your own central repository.

### 2. Add Workflow to Your Repos

Create `.github/workflows/release.yml` in each repository:

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
    uses: YOUR-ORG/release-dispatcher-action/.github/workflows/release.yml@main
    with:
      dry-run: ${{ inputs.dry_run || false }}
    secrets: inherit
```

### 3. Configure Secrets

Add these secrets to your organization or repositories:

| Secret | Required | Description |
|--------|----------|-------------|
| `APP_ID` | Yes | GitHub App ID |
| `APP_PRIVATE_KEY` | Yes | GitHub App private key (PEM) |
| `ANTHROPIC_API_KEY` | No | For Claude-powered PR descriptions |
| `CLAUDE_REVIEW_PAT` | No | For Claude code review |
| `SAVVYWEB_NPM_AUTH_TOKEN` | No | For custom registry publishing |

### 4. Done!

Your repos now have:
- Automated release branch management
- PR validation (title, commits, lint, tests)
- npm publishing with provenance
- GitHub release creation

## Workflow Phases

| Phase | Trigger | What Happens |
|-------|---------|---------------|
| **branch-management** | Push to main | Creates/updates `changeset-release/main` branch with version bumps |
| **validation** | PR opened/updated | Runs PR title, commit, lint, and test checks |
| **publishing** | Release PR merged | Publishes packages to npm with provenance, creates GitHub releases |
| **close-issues** | Release PR merged | Closes issues linked in commits |

## Dependencies

This dispatcher uses these companion actions:

| Action | Purpose |
|--------|----------|
| [workflow-control-action](https://github.com/savvy-web/workflow-control-action) | Detects which phase to run |
| [workflow-runtime-action](https://github.com/savvy-web/workflow-runtime-action) | Sets up Node.js, pnpm, etc. |
| [workflow-release-action](https://github.com/savvy-web/workflow-release-action) | Executes release logic |

You'll need to fork/copy these as well, or use the Savvy Web versions.

## Customization

See [CUSTOMIZATION.md](./docs/CUSTOMIZATION.md) for:
- Custom registry configuration
- Validation check customization  
- Branch naming conventions
- Monorepo support

## Troubleshooting

See [TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md) for common issues.

## License

MIT
