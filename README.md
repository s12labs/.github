# s12labs/.github - Organization-Wide Workflows

This repository contains reusable GitHub Actions workflows and templates for the s12labs organization.

## Overview

The workflows in this repository enable **zero-touch deployment** for Go applications to the nugget Kubernetes cluster. New apps automatically get:

- Docker image builds (optimized with Blacksmith for 40x faster builds)
- GitOps-based deployments via Flux
- Kubernetes manifests (namespace, deployment, service, ingress)
- Tailscale ingress for private access

## Reusable Workflows

### 1. Docker Build (`reusable-docker-build.yml`)

Builds and pushes Docker images to GitHub Container Registry with Blacksmith optimizations.

**Usage:**

```yaml
jobs:
  build:
    uses: s12labs/.github/.github/workflows/reusable-docker-build.yml@main
    with:
      app_name: my-api  # Optional, defaults to repository name
      dockerfile_path: ./Dockerfile  # Optional
      context: .  # Optional
      platforms: linux/amd64  # Optional
    secrets: inherit
```

**Inputs:**
- `app_name` (optional): Application name, defaults to repository name
- `dockerfile_path` (optional): Path to Dockerfile, default `./Dockerfile`
- `context` (optional): Build context path, default `.`
- `platforms` (optional): Target platforms, default `linux/amd64`

**Outputs:**
- `image_tag`: The version tag (e.g., `v0.1.0`)
- `image_digest`: SHA256 digest of the built image

**Features:**
- Uses Blacksmith runners (`blacksmith-2vcpu`) for 1.5-2x faster execution
- Persistent Docker layer cache (40x faster builds on unchanged layers)
- Automatic tagging (semver, sha, branch, latest)
- Pushes to `ghcr.io/s12labs/{app_name}`
- Injects build args: `VERSION`, `COMMIT`, `BUILD_TIME`

**Secrets Required:**
- `BLACKSMITH_API_KEY` (optional): For Blacksmith optimizations
- `GITHUB_TOKEN` (automatic): For GHCR authentication

### 2. GitOps Update (`reusable-gitops-update.yml`)

Updates the compute-gitops repository with new image tags and optionally bootstraps new applications.

**Usage:**

```yaml
jobs:
  deploy:
    uses: s12labs/.github/.github/workflows/reusable-gitops-update.yml@main
    with:
      app_name: my-api
      image_tag: v0.1.0
      cluster: nugget  # Optional
      bootstrap_if_missing: true  # Optional
    secrets:
      GITOPS_PAT: ${{ secrets.GITOPS_PAT }}
```

**Inputs:**
- `app_name` (required): Application name (must match directory in compute-gitops)
- `image_tag` (required): Image tag to deploy (e.g., `v0.1.0`)
- `cluster` (optional): Target cluster name, default `nugget`
- `bootstrap_if_missing` (optional): Create manifests if they don't exist, default `true`

**Secrets Required:**
- `GITOPS_PAT`: Personal Access Token with write access to `s12labs/compute-gitops`

**Features:**
- Checks if app manifests exist in compute-gitops
- If missing and `bootstrap_if_missing=true`: runs `bootstrap-app.sh` to generate manifests
- Updates `deployment.yaml` with new image tag
- Commits and pushes to compute-gitops
- Respects `# auto-update: disabled` comment in deployment files

## Workflow Templates

### Go API Release and Deploy (`go-api-release.yml`)

Starter workflow template for new Go API repositories. This template appears in the GitHub UI when creating a new workflow.

**What it does:**

1. **On push to main**: Runs Release Please to create/update release PR
2. **On release published**:
   - Builds Docker image
   - Updates GitOps manifests
   - Deploys to nugget cluster

**Setup for new repositories:**

1. Create repository from `s12labs/go-api-template`
2. Add required secrets:
   ```bash
   gh secret set RELEASE_PLEASE_TOKEN --body "$TOKEN"
   gh secret set GITOPS_PAT --body "$PAT"
   gh secret set BLACKSMITH_API_KEY --body "$KEY"  # Optional
   ```
3. Workflow file is already included in template
4. Push code to main
5. Merge Release Please PR to trigger deployment

## Performance Benefits

### Blacksmith Optimizations

- **Runners:** `blacksmith-2vcpu` provides 1.5-2x faster execution than standard runners
- **Docker Cache:** Persistent NVMe-backed layer cache shared across all CI runs
- **Build Speed:** 40x faster on unchanged layers (seconds instead of minutes)
- **Free Tier:** 3,000 minutes/month on 2vcpu runners

### Traditional vs. Optimized Build Times

| Scenario | Traditional | With Blacksmith |
|----------|-------------|-----------------|
| First build | 3-5 minutes | 3-5 minutes |
| Code-only change | 3-5 minutes | 10-30 seconds |
| Dependency change | 3-5 minutes | 1-2 minutes |

## Resource Profiles

All applications use lean resource limits by default to prevent cluster exhaustion:

### API (Default)
```yaml
requests: {cpu: 250m, memory: 256Mi}
limits: {cpu: 1000m, memory: 1Gi}
```

### Web
```yaml
requests: {cpu: 500m, memory: 512Mi}
limits: {cpu: 1000m, memory: 2Gi}
```

### Worker
```yaml
requests: {cpu: 250m, memory: 512Mi}
limits: {cpu: 2000m, memory: 2Gi}
```

## Required Organization Secrets

Set these at the organization level for all repositories to use:

```bash
# Create a PAT with repo and packages:write scopes
gh secret set GITOPS_PAT --org s12labs --body "$PAT"

# Create a PAT with repo scope for Release Please
gh secret set RELEASE_PLEASE_TOKEN --org s12labs --body "$TOKEN"

# Optional: Blacksmith API key for optimized builds
gh secret set BLACKSMITH_API_KEY --org s12labs --body "$KEY"
```

## Architecture

```
Developer Flow:
  1. Use go-api-template → my-new-api
  2. Write Go code
  3. Push to main → Release Please creates PR
  4. Merge PR → v0.1.0 tag
  5. Automatic: Build → Push → GitOps Update → Deploy

Platform Layers:
  Layer 1: Reusable workflows (this repo)
  Layer 2: Kustomize base templates (compute-gitops/base/)
  Layer 3: Template repository (go-api-template)
```

## Troubleshooting

### Build failures

Check workflow run logs for Docker build errors. Common issues:
- Invalid Dockerfile syntax
- Missing dependencies in go.mod
- Build context path incorrect

### GitOps update failures

- Verify `GITOPS_PAT` has write access to compute-gitops
- Check if bootstrap script executed successfully
- Ensure app name matches between workflow and manifests

### Deployment not appearing

- Wait 1-2 minutes for Flux reconciliation
- Check Flux logs: `flux logs -n flux-system`
- Verify manifests in compute-gitops are valid
- Check pod status: `kubectl get pods -n {app-name}`

## Contributing

To modify reusable workflows:

1. Make changes in this repository
2. Test with a sample app repository
3. Update documentation in this README
4. Commit and push to main
5. Existing workflows will use updated version on next run

## Support

For issues or questions:
- Check the [compute-gitops ONBOARDING.md](https://github.com/s12labs/compute-gitops/blob/main/ONBOARDING.md)
- Review workflow run logs in GitHub Actions
- Contact the platform team
