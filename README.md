# github-workflows

Shared reusable GitHub Actions workflows for the krystof-io organization.

## Available Workflows

| Workflow | Purpose |
|----------|---------|
| `java-app-build.yml` | Build, test, and deploy Java Spring Boot applications |
| `java-lib-build.yml` | Build, test, and publish Java libraries |
| `maven-release.yml` | Create Maven releases with version tags |
| `sonar-scan.yml` | Run SonarQube analysis and OWASP dependency checks |
| `flux-deploy.yml` | Deploy applications via GitOps/Flux |
| `validate-flux-deployment.yml` | Validate Kubernetes deployments |

## Quick Start

Add three workflow files to your application repository:

**`.github/workflows/build.yml`** - Build and deploy to dev:
```yaml
name: Build and Dev Deploy

on:
  push:
    branches: ["*"]
    tags-ignore: ['v*']
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write

jobs:
  build-and-dev-deploy:
    uses: krystof-io/github-workflows/.github/workflows/java-app-build.yml@main
    with:
      app_name: my-service
      target_clusters: '["private"]'
    secrets: inherit
```

**`.github/workflows/promote.yml`** - Deploy to prod on version tags:
```yaml
name: Prod Promotion

on:
  push:
    tags: ['v*']

permissions:
  contents: read
  pull-requests: write

jobs:
  promote-to-prod:
    uses: krystof-io/github-workflows/.github/workflows/java-app-build.yml@main
    with:
      app_name: my-service
      target_clusters: '["private"]'
    secrets: inherit
```

**`.github/workflows/release.yml`** - Create Maven releases (manual trigger):
```yaml
name: Maven Release

on:
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Perform a dry run'
        type: boolean
        default: false

permissions:
  contents: write
  pull-requests: write

jobs:
  maven-release:
    uses: krystof-io/github-workflows/.github/workflows/maven-release.yml@main
    with:
      dry_run: ${{ github.event.inputs.dry_run == 'true' }}
    secrets: inherit
```

## Cluster Configuration

Clusters are defined in `config/clusters.yml`. Currently configured:

- **private** - Private infrastructure cluster (`dev` and `prod` namespaces)

## Documentation

- **[Setup Guide](samples/README.md)** - Detailed setup instructions for the CI/CD pipeline
- **[CLAUDE.md](CLAUDE.md)** - Architecture reference for AI assistants

## Sample Workflows

See the `samples/` directory for example workflow configurations:

**For Spring Boot applications:**
- `build.yml` - Build and dev deploy
- `promote.yml` - Production promotion on version tags
- `release.yml` - Maven release (manual trigger)

**For Java libraries:**
- `ci-cd-lib.yml` - Build, test, and publish

**For GitOps repositories:**
- `validate-pr.yml` - PR validation for deployment repos
