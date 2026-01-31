# GitHub App Authentication Setup

This guide explains how to configure GitHub App authentication for the s12labs deployment platform (recommended over Personal Access Tokens).

## Why GitHub App Instead of PATs?

- ✅ **More secure** - Scoped permissions, short-lived tokens
- ✅ **Organization-owned** - Not tied to a personal account
- ✅ **Better audit trail** - Actions show as the app, not a user
- ✅ **Auto token refresh** - No token expiration management
- ✅ **Fine-grained permissions** - Only what's needed

## Prerequisites

You need an existing GitHub App with these permissions:

- **Repository > Contents**: Read & Write (to update compute-gitops)
- **Repository > Pull Requests**: Read & Write (for Release Please)
- **Repository > Metadata**: Read (automatic)

## Setup Instructions

### Step 1: Get GitHub App Credentials

1. Go to your GitHub App settings:
   - Organization: https://github.com/organizations/s12labs/settings/apps
   - Or: https://github.com/settings/apps (for personal apps)

2. Click on your GitHub App (e.g., "s12labs-automation")

3. Note the **App ID** (you'll need this)

4. Generate a **Private Key**:
   - Scroll to "Private keys" section
   - Click "Generate a private key"
   - Save the downloaded `.pem` file

### Step 2: Set Organization-Level Secrets (Recommended)

**Option A: Via GitHub CLI**

```bash
# Set App ID
gh secret set GH_APP_ID --org s12labs --visibility all --body "123456"

# Set Private Key (from the .pem file)
gh secret set GH_APP_PRIVATE_KEY --org s12labs --visibility all < path/to/private-key.pem
```

**Option B: Via GitHub Web UI**

1. Go to: https://github.com/organizations/s12labs/settings/secrets/actions
2. Click "New organization secret"
3. Create `GH_APP_ID`:
   - Name: `GH_APP_ID`
   - Value: Your App ID (e.g., `123456`)
   - Repository access: "All repositories" or "Selected repositories"
4. Create `GH_APP_PRIVATE_KEY`:
   - Name: `GH_APP_PRIVATE_KEY`
   - Value: Paste the entire contents of your `.pem` file
   - Repository access: Same as above

### Step 3: Install the App to Repositories

1. Go to: https://github.com/organizations/s12labs/settings/installations
2. Click "Configure" next to your GitHub App
3. Under "Repository access", select:
   - "All repositories" (easiest)
   - Or "Only select repositories" and add:
     - `s12labs/compute-gitops`
     - `s12labs/go-api-template`
     - Any app repositories you create

### Step 4: Verify Permissions

Ensure your GitHub App has these permissions:

```yaml
Permissions:
  Contents: Read & Write
  Pull Requests: Read & Write
  Metadata: Read (automatic)
```

To update permissions:
1. Go to app settings
2. Click "Permissions & events"
3. Update permissions
4. Save changes
5. Organization owners will need to approve the changes

## Setting Repository-Level Secrets (Alternative)

If you don't have org admin access, set secrets per repository:

```bash
cd /path/to/your-app-repo

# Set App ID
gh secret set GH_APP_ID --body "123456"

# Set Private Key
gh secret set GH_APP_PRIVATE_KEY < path/to/private-key.pem
```

## Workflow Behavior

The updated workflows support **both** authentication methods with this priority:

1. **GitHub App** (preferred) - Uses `GH_APP_ID` + `GH_APP_PRIVATE_KEY`
2. **PAT** (fallback) - Uses `RELEASE_PLEASE_TOKEN` or `GITOPS_PAT`
3. **GITHUB_TOKEN** (limited) - Only for Release Please, limited permissions

### Example Workflow Logs

**With GitHub App:**
```
Generate GitHub App Token
✓ Token generated successfully

Determine authentication token
Using GitHub App authentication

Checkout compute-gitops repository
✓ Cloned using app token
```

**With PAT (deprecated):**
```
Generate GitHub App Token
⊘ Skipped (secrets not set)

Determine authentication token
Using PAT authentication (deprecated)

Checkout compute-gitops repository
✓ Cloned using PAT
```

## Migration from PATs

If you're currently using PATs, you can migrate gradually:

### Phase 1: Add GitHub App Secrets
Add `GH_APP_ID` and `GH_APP_PRIVATE_KEY` secrets while keeping existing PATs.
Workflows will automatically use GitHub App authentication.

### Phase 2: Test
Create a test deployment to verify GitHub App authentication works.

### Phase 3: Remove PATs
Once verified, you can remove the old PAT secrets:
```bash
gh secret delete RELEASE_PLEASE_TOKEN --org s12labs
gh secret delete GITOPS_PAT --org s12labs
```

## Troubleshooting

### "Resource not accessible by integration"

**Cause:** GitHub App doesn't have required permissions.

**Solution:**
1. Check app permissions include "Contents: Write" and "Pull Requests: Write"
2. Ensure app is installed to the target repositories
3. Re-approve permissions if recently updated

### "Bad credentials"

**Cause:** Private key is incorrect or malformed.

**Solution:**
1. Verify you copied the entire `.pem` file including:
   ```
   -----BEGIN RSA PRIVATE KEY-----
   ...
   -----END RSA PRIVATE KEY-----
   ```
2. No extra spaces or newlines at start/end
3. Re-generate private key if needed

### Workflows still using PAT

**Cause:** GitHub App secrets not set or workflow using old version.

**Solution:**
1. Verify `GH_APP_ID` and `GH_APP_PRIVATE_KEY` secrets are set
2. Check workflow is using `s12labs/.github/.github/workflows/reusable-gitops-update.yml@main`
3. Force workflow to re-run: `gh run rerun <run-id>`

### Token has insufficient permissions

**Cause:** Repository not included in app installation.

**Solution:**
1. Go to: https://github.com/organizations/s12labs/settings/installations
2. Click "Configure" on your app
3. Add missing repositories to "Repository access"

## Security Best Practices

1. **Limit repository access** - Only install app to repositories that need it
2. **Use organization secrets** - Easier to manage and rotate
3. **Rotate keys regularly** - Generate new private key every 90 days
4. **Audit app usage** - Review app activity in GitHub audit log
5. **Minimal permissions** - Only grant permissions actually needed

## Reference

- [GitHub Apps Documentation](https://docs.github.com/en/apps)
- [Creating GitHub App tokens in workflows](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#using-the-github_token-in-a-workflow)
- [actions/create-github-app-token](https://github.com/actions/create-github-app-token)

## Next Steps

After setting up GitHub App authentication:

1. Test with a new repository created from `go-api-template`
2. Verify workflows use GitHub App in logs
3. Remove PAT secrets (optional)
4. Update ONBOARDING.md to reference GitHub App setup
