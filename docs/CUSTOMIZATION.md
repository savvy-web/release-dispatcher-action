# Customization Guide

## Custom Registries

To publish to custom npm registries (e.g., private Verdaccio, Artifactory):

### 1. Add Registry Token Secret

```bash
gh secret set CUSTOM_NPM_TOKEN --org YOUR-ORG --body "your-token"
```

### 2. Update Dispatcher Workflows

In `release-branch.yml` and `release-publish.yml`, add the custom registry:

```yaml
- name: Run release
  uses: YOUR-ORG/workflow-release-action@main
  with:
    # ... other inputs
    custom-registries: |
      https://registry.example.com/${{ secrets.CUSTOM_NPM_TOKEN }}
```

### 3. Configure Package

```json
{
  "publishConfig": {
    "registry": "https://registry.example.com/"
  }
}
```

## Branch Naming

Default release branch: `changeset-release/main`

To customize, update the workflow-control-action configuration:

```yaml
- name: Detect workflow phase
  id: control
  uses: YOUR-ORG/workflow-control-action@main
  with:
    release-branch: custom-release/main
    target-branch: main
```

## Validation Customization

### Disable Specific Checks

Fork `release-validate.yml` and remove unwanted checks:

```yaml
# Remove or comment out checks you don't want
- name: Run Biome Lint
  id: lint
  # ...
```

### Custom Test Command

The default is `pnpm ci:test`. To customize, update your `package.json`:

```json
{
  "scripts": {
    "ci:test": "your-test-command"
  }
}
```

### Skip Validation for Bot PRs

Add a condition to the validation workflow:

```yaml
jobs:
  validate:
    if: github.actor != 'dependabot[bot]'
    # ...
```

## Claude Integration

### PR Description Generation

Set `ANTHROPIC_API_KEY` to enable Claude-generated PR descriptions:

```bash
gh secret set ANTHROPIC_API_KEY --org YOUR-ORG --body "sk-ant-..."
```

### Code Review

Set `CLAUDE_REVIEW_PAT` for Claude code review integration:

```bash
gh secret set CLAUDE_REVIEW_PAT --org YOUR-ORG --body "ghp_..."
```

## Per-Repo Overrides

For repos that need different behavior, you can:

### Option 1: Override Inputs

```yaml
jobs:
  release:
    uses: YOUR-ORG/release-dispatcher-action/.github/workflows/release.yml@main
    with:
      dry-run: true  # Always dry-run for this repo
    secrets: inherit
```

### Option 2: Use Local Workflows

Copy the workflow files locally and customize:

```yaml
jobs:
  release:
    uses: ./.github/workflows/release-custom.yml
    secrets: inherit
```

### Option 3: Pin to Specific Version

```yaml
jobs:
  release:
    uses: YOUR-ORG/release-dispatcher-action/.github/workflows/release.yml@v1.0.0
    secrets: inherit
```

## Adding New Checks

To add custom validation checks:

1. Fork `release-validate.yml`
2. Add your check:

```yaml
- name: Custom Security Scan
  id: security
  continue-on-error: true
  run: |
    set +e
    npm audit --audit-level=high
    echo "exit_code=$?" >> $GITHUB_OUTPUT

- name: Update Security Check
  if: always()
  uses: actions/github-script@v8
  with:
    github-token: ${{ steps.app-token.outputs.token }}
    script: |
      const ok = '${{ steps.security.outputs.exit_code }}' === '0';
      await github.rest.checks.update({
        owner: context.repo.owner,
        repo: context.repo.repo,
        check_run_id: process.env.SECURITY_CHECK_ID,
        status: 'completed',
        conclusion: ok ? 'success' : 'failure',
        output: { 
          title: ok ? 'No vulnerabilities' : 'Vulnerabilities found',
          summary: ok ? 'Security scan passed' : 'Found security issues'
        }
      });
```

## Notifications

To add Slack/Discord notifications on release:

```yaml
# Add to release-publish.yml after the publish job
notify:
  name: Notify
  needs: publish
  if: success()
  runs-on: ubuntu-latest
  steps:
    - name: Notify Slack
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "New release published!"
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```
