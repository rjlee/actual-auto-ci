# actual-auto-ci

Shared GitHub Actions, composite actions, and scripts used across the `actual-*` service repositories. Centralising workflows keeps CI/CD consistent (Node version, lint/test steps, release automation, Docker publishing).

## Contents

- `.github/workflows/ci.yml` – reusable workflow for linting, format checking, and tests.
- `.github/workflows/release.yml` – reusable workflow invoking semantic-release.
- `.github/workflows/docker-api-matrix.yml` – reusable workflow building API-pinned Docker images.
- `.github/workflows/docker-release.yml` – reusable workflow building and pushing images for `latest`/stable.
- `.github/actions/*` – composite actions (`setup-node`, `compute-api-versions`, `docker-tags`) shared by workflows.

## Requirements

- GitHub Actions runners (tested on `ubuntu-latest`).
- Node.js 20/22+ (configurable per caller).
- Conventional commits (semantic-release uses commit messages to determine next version).
- Access to GHCR (for Docker workflows) and optional npm credentials if publishing to npm (currently `npmPublish: false`).

## Usage

### CI workflow

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: your-org/actual-auto-ci/.github/workflows/ci.yml@v1
    with:
      node-version: '22'
      package-manager: npm    # npm | pnpm | yarn
      lint: true
      format-check: true
      test: true
```

### Release workflow

```yaml
name: Release
on:
  workflow_run:
    workflows: ['CI']
    types: [completed]
  workflow_dispatch:

jobs:
  release:
    if: ${{ github.event_name == 'workflow_dispatch'
          || github.event.workflow_run.conclusion == 'success' }}
    uses: your-org/actual-auto-ci/.github/workflows/release.yml@v1
    with:
      node-version: '22'
      package-manager: npm
```

### Docker API matrix

Builds images for the latest patch in the last N API majors:

```yaml
name: Docker Build API Matrix
on:
  workflow_run:
    workflows: ['Release']
    types: [completed]
  schedule:
    - cron: '0 6 * * 1'

jobs:
  docker:
    uses: your-org/actual-auto-ci/.github/workflows/docker-api-matrix.yml@v1
    with:
      image-name: ghcr.io/${{ github.repository }}
      dockerfile: Dockerfile
      context: .
      platforms: linux/amd64,linux/arm64
      majors-window: 3
      include-stable: true
```

### Composite actions

- `setup-node` – installs Node with caching for the configured package manager.
- `compute-api-versions` – finds the latest patch for the last N stable `@actual-app/api` majors.
- `docker-tags` – expands a base tag into major/minor aliases and optional `api-stable`.

Callers can import them directly:

```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: your-org/actual-auto-ci/.github/actions/setup-node@v1
        with:
          node-version: '22'
          package-manager: npm
```

## Versioning

Workflows and actions are tagged (`v1`, `v1.1.0`, etc.). Pin callers to a major tag (`@v1`) and use Dependabot/Renovate to adopt new releases.

## Local development

- `npm install` – sets up linting scripts for the workflow sources.
- `npm test` – ensures scripts/composite action helpers pass unit checks (if present).
- Edit workflow YAML and composite actions under `.github/`.

## Release process

1. Merge changes to `main`.
2. Tag the repository (`git tag v1.2.0 && git push origin v1.2.0`).
3. Update consuming repos to the new tag.

## License

MIT © contributors.
