# actual-auto-ci

Shared GitHub Actions for the `actual-*` projects. These workflows are reusable via `workflow_call` so each project can stay DRY while keeping local logic minimal.

## Workflows

- CI (`.github/workflows/ci.yml`): Lint, format-check, and tests.
- Release (`.github/workflows/release.yml`): Runs semantic-release.
- Docker API Matrix (`.github/workflows/docker-api-matrix.yml`): Builds and publishes Docker images for the latest patch in the last N stable `@actual-app/api` majors, tagging each build with its exact semver plus `latest` (highest stable).

## Composite Actions

- `compute-api-versions`: Determines the latest patch for the last N stable `@actual-app/api` majors.
- `docker-tags`: Composes tag lists for a specific API version (exact semver) and optionally adds `latest`.
- `setup-node`: Installs a specific Node version with caching for the selected package manager.

## Usage (caller repos)

Example CI caller:

```yaml
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: <org>/actual-auto-ci/.github/workflows/ci.yml@v1
    with:
      node-version: '20'
      package-manager: npm
      lint: true
      format-check: true
      test: true
```

Example Release caller:

```yaml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    uses: <org>/actual-auto-ci/.github/workflows/release.yml@v1
    with:
      node-version: '20'
      package-manager: npm
```

Example Docker API matrix caller:

```yaml
name: Docker Build API Matrix
on:
  workflow_dispatch:
  workflow_run:
    workflows: ['CI']
    types: [completed]
  schedule:
    - cron: '0 6 * * 1'
jobs:
  docker:
    uses: <org>/actual-auto-ci/.github/workflows/docker-api-matrix.yml@v1
    with:
      image-name: ghcr.io/${{ github.repository }}
      platforms: linux/amd64,linux/arm64
      dockerfile: Dockerfile
      context: .
```

Pin callers to `@v1` and use Renovate/Dependabot to roll forward when ready.
