# Implementation Guide

This guide walks you through setting up the release dispatcher for your organization.

## Prerequisites

### 1. GitHub App

Create a GitHub App for your organization with these permissions:

**Repository Permissions:**
- Actions: Read
- Checks: Read & Write
- Contents: Read & Write
- Issues: Read & Write
- Pull Requests: Read & Write
- Packages: Write (if using GitHub Packages)

**Organization Permissions:**
- Members: Read (optional, for team mentions)

**Subscribe to Events:**
- Check run
- Check suite
- Pull request
- Push

### 2. Changesets

Your repositories should use [changesets](https://github.com/changesets/changesets) for version management:

```bash
pnpm add -D @changesets/cli
pnpm changeset init
```

### 3. Package Configuration

Ensure your `package.json` has:

```json
{
  "name": "@your-org/package-name",
  "version": "1.0.0",
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org/"
  }
}
```

## Step-by-Step Setup

### Step 1: Create the Dispatcher Repository

```bash
# Option A: Fork from Savvy Web
gh repo fork savvy-web/release-dispatcher-action --org YOUR-ORG

# Option B: Create new and copy files
gh repo create YOUR-ORG/release-dispatcher-action --public
# Then copy the .github/workflows/ files
```

### Step 2: Fork/Copy Companion Actions

You'll also need these actions:

```bash
# Phase detection
gh repo fork savvy-web/workflow-control-action --org YOUR-ORG

# Runtime setup (Node.js, pnpm, etc.)
gh repo fork savvy-web/workflow-runtime-action --org YOUR-ORG

# Release logic
gh repo fork savvy-web/workflow-release-action --org YOUR-ORG
```

### Step 3: Update Action References

In your forked `release-dispatcher-action`, update the workflow files to reference your org:

```yaml
# Before
uses: savvy-web/workflow-control-action@main
uses: savvy-web/workflow-runtime-action@main
uses: savvy-web/workflow-release-action@main

# After
uses: YOUR-ORG/workflow-control-action@main
uses: YOUR-ORG/workflow-runtime-action@main
uses: YOUR-ORG/workflow-release-action@main
```

### Step 4: Configure Organization Secrets

Add secrets at the organization level for automatic inheritance:

```bash
# GitHub App credentials
gh secret set APP_ID --org YOUR-ORG --body "123456"
gh secret set APP_PRIVATE_KEY --org YOUR-ORG < private-key.pem

# Optional: Claude integration
gh secret set ANTHROPIC_API_KEY --org YOUR-ORG --body "sk-ant-..."
gh secret set CLAUDE_REVIEW_PAT --org YOUR-ORG --body "ghp_..."

# Optional: Custom registry
gh secret set SAVVYWEB_NPM_AUTH_TOKEN --org YOUR-ORG --body "npm_..."
```

### Step 5: Add Workflow to Each Repository

Create `.github/workflows/release.yml`:

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

### Step 6: Test the Setup

1. Create a changeset:
   ```bash
   pnpm changeset
   ```

2. Commit and push:
   ```bash
   git add .
   git commit -m "feat: add new feature"
   git push
   ```

3. Watch the workflow:
   ```bash
   gh run watch
   ```

4. A release PR should be created automatically.

## npm Provenance

For npm provenance (supply chain attestation), ensure:

1. **id-token: write** permission is set
2. **attestations: write** permission is set
3. Your npm account has provenance enabled
4. The publishing workflow runs on GitHub Actions (not self-hosted)

The `release-publish.yml` workflow includes these permissions by default.

## Monorepo Support

For monorepos with multiple packages:

1. Configure changesets for monorepo:
   ```json
   // .changeset/config.json
   {
     "changelog": "@changesets/cli/changelog",
     "commit": false,
     "fixed": [],
     "linked": [],
     "access": "public",
     "baseBranch": "main",
     "updateInternalDependencies": "patch"
   }
   ```

2. Each package gets its own version bump and publish.

3. GitHub releases are created per-package.

## Validation Checks

The dispatcher runs these checks on PRs:

| Check | Tool | Customization |
|-------|------|---------------|
| PR Title | commitlint | Configure in `commitlint.config.ts` |
| Commit Messages | commitlint | Configure in `commitlint.config.ts` |
| Code Quality | Biome | Configure in `biome.jsonc` |
| Tests | pnpm ci:test | Configure test script in `package.json` |

## Next Steps

- [Customization Guide](./CUSTOMIZATION.md)
- [Troubleshooting](./TROUBLESHOOTING.md)
