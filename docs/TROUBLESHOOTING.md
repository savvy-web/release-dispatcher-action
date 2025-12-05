# Troubleshooting Guide

## Common Issues

### "startup_failure" Error

**Cause:** Usually a permissions issue or invalid workflow syntax.

**Solutions:**
1. Ensure the dispatcher workflow has all required permissions:
   ```yaml
   permissions:
     contents: write
     pull-requests: write
     id-token: write
     packages: write
     attestations: write
     checks: write
   ```

2. Validate workflow syntax:
   ```bash
   gh workflow view release.yml
   ```

### Workflow Not Triggering

**Cause:** Incorrect trigger configuration.

**Solutions:**
1. Verify the `on:` triggers match your branch names:
   ```yaml
   on:
     push:
       branches: [main]  # Must match your default branch
     pull_request:
       branches: [main, changeset-release/main]
   ```

2. Check if the workflow file is on the default branch.

### "Resource not accessible by integration"

**Cause:** GitHub App missing permissions.

**Solutions:**
1. Update GitHub App permissions (see Implementation Guide)
2. Reinstall the app on your repository
3. Check that secrets are correctly set:
   ```bash
   gh secret list --org YOUR-ORG
   ```

### Phase Detection Not Working

**Cause:** workflow-control-action not detecting the correct phase.

**Solutions:**
1. Check the "Detect Phase" job output in the workflow run
2. Verify branch names match configuration:
   ```yaml
   - uses: YOUR-ORG/workflow-control-action@main
     with:
       release-branch: changeset-release/main
       target-branch: main
   ```

### npm Publish Failing

**Cause:** Authentication or provenance issues.

**Solutions:**
1. For npm provenance, ensure `id-token: write` permission
2. Check npm token is valid:
   ```bash
   npm whoami
   ```
3. Verify package.json publishConfig:
   ```json
   {
     "publishConfig": {
       "access": "public",
       "registry": "https://registry.npmjs.org/"
     }
   }
   ```

### Validation Checks Stuck "in_progress"

**Cause:** Workflow cancelled before cleanup job ran.

**Solutions:**
1. The cleanup job should resolve orphaned checks
2. Manually complete checks via API:
   ```bash
   gh api repos/OWNER/REPO/check-runs/CHECK_ID \
     -X PATCH \
     -f status=completed \
     -f conclusion=neutral
   ```

### Secrets Not Inherited

**Cause:** `secrets: inherit` not working across repos.

**Solutions:**
1. Ensure secrets are set at organization level
2. Check repository has access to org secrets
3. Explicitly pass secrets if needed:
   ```yaml
   jobs:
     release:
       uses: YOUR-ORG/release-dispatcher-action/.github/workflows/release.yml@main
       secrets:
         APP_ID: ${{ secrets.APP_ID }}
         APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }}
   ```

### Release PR Not Created

**Cause:** No changesets or branch-management phase skipped.

**Solutions:**
1. Verify changesets exist:
   ```bash
   ls .changeset/*.md
   ```
2. Check phase detection output - should be "branch-management"
3. Ensure push is to the target branch (usually `main`)

### Multiple Workflow Runs

**Cause:** Multiple events triggering the workflow.

**Solutions:**
1. This is expected - the dispatcher handles all events
2. Each run should only execute the relevant phase
3. No concurrency issues since phases are mutually exclusive

## Debugging

### Enable Debug Logging

```bash
# Set repository variable
gh variable set ACTIONS_RUNNER_DEBUG --body "true" --repo OWNER/REPO
gh variable set ACTIONS_STEP_DEBUG --body "true" --repo OWNER/REPO
```

### View Workflow Logs

```bash
# List recent runs
gh run list --limit 5

# View specific run
gh run view RUN_ID --log

# Watch live
gh run watch RUN_ID
```

### Check Phase Detection

The control job outputs these values:
- `phase`: The detected phase (branch-management, validation, publishing, etc.)
- `is_pr_event`: Whether this is a PR event
- `is_pr_merged`: Whether the PR was merged
- `has_changesets`: Whether changeset files exist

```bash
gh run view RUN_ID --json jobs | jq '.jobs[] | select(.name | contains("Detect"))'
```

## Getting Help

1. Check the workflow run logs for detailed error messages
2. Review the [Implementation Guide](./IMPLEMENTATION.md)
3. Open an issue on the dispatcher repository
