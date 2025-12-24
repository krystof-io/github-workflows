# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a **shared GitHub Actions workflows repository** for the krystof-io organization. It provides reusable CI/CD workflows that other repositories call via `workflow_call`. Application and library repos only need a minimal workflow file that references these shared workflows.

## Key Workflows

### `.github/workflows/java-app-build.yml`
Main workflow for Java Spring Boot applications. Handles:
- Maven build, test, and package
- Docker image build and push (uses Dockerfiles from `krystof-io/devops-templates`)
- SonarQube analysis (via `sonar-scan.yml`)
- GitOps deployment via Flux (via `flux-deploy.yml`)
- Deployment validation (via `validate-flux-deployment.yml`)

Called by app repos with `uses: krystof-io/github-workflows/.github/workflows/java-app-build.yml@main`

### `.github/workflows/java-lib-build.yml`
Workflow for Java libraries. Handles:
- Maven build and test
- SonarQube analysis
- SNAPSHOT publishing (automatic for SNAPSHOT versions)
- Release publishing (requires passing SonarQube, triggered by `v*` tags)

### `.github/workflows/maven-release.yml`
Creates Maven releases by running `mvn release:prepare`. Pushes version tags that trigger deployments.

### `.github/workflows/sonar-scan.yml`
Runs SonarQube analysis and OWASP dependency checks. Outputs quality gate status, metrics, and dashboard URLs. Skipped when commit message contains "prepare for next development iteration".

### `.github/workflows/flux-deploy.yml`
Updates GitOps repositories with new image tags. Creates PRs that are auto-merged for deployments.

### `.github/workflows/validate-flux-deployment.yml`
Validates Kubernetes deployments after Flux reconciliation. Checks that correct image tags are running in pods.

## Cluster Configuration

`config/clusters.yml` maps cluster names to their GitOps repositories. Workflows read this to determine deployment targets.

```yaml
clusters:
  private:
    gitops_repo: 'krystof-io/kio-private-cluster'
    cluster_path: 'k8s-manifests/components/apps/workloads'
    cluster_url: 'https://private.krystof.io:6443'
```

To add a new cluster, add an entry here. Apps can then target it with `target_clusters: '["new-cluster-name"]'`.

## Branching and Deployment Strategy

- **Feature branches** (`feature/*`, `feat/*`, `bugfix/*`, `fix/*`): Deploy to `dev` namespace with feature-specific image tags
- **Main branch**: Deploy to `dev` namespace with SNAPSHOT versions
- **Tags** (`v*`): Deploy to `prod` namespace via auto-merged PR

Namespaces: `dev` and `prod` (previously `kio-dev` and `kio-prod`)

## Required Secrets (Organization Level)

Workflows expect these secrets to be available:
- `IMAGE_REGISTRY_USERNAME`, `IMAGE_REGISTRY_PASSWORD` - Docker registry credentials
- `MAVEN_REPO_USERNAME`, `MAVEN_REPO_PASSWORD` - Nexus Maven repository credentials
- `SONAR_TOKEN` - SonarQube authentication
- `NVD_API_KEY` - For OWASP dependency-check
- `S3_CACHE_ACCESS_KEY`, `S3_CACHE_SECRET_KEY` - Maven cache (S3-compatible)
- `CICD_TOKEN` - GitHub PAT for GitOps repo access
- `CICD_USER_NAME`, `CICD_USER_EMAIL` - Git commit identity
- Cluster-specific: `K8S_CLUSTER_CA_CERT_FOR_{CLUSTER}_CLUSTER`, `SERVICE_ACCOUNT_TOKEN_FOR_{CLUSTER}_CLUSTER_CICD_K8S_ACCESS`

## Required Variables (Organization Level)

- `IMAGE_REGISTRY_HOST` - Docker registry hostname
- `MAVEN_REPO_URL` - Maven public repository URL
- `MAVEN_PRIVATE_RELEASE_REPO_URL`, `MAVEN_PRIVATE_SNAPSHOT_REPO_URL` - Maven deploy URLs
- `SONAR_HOST_URL` - SonarQube server URL
- `S3_CACHE_ENDPOINT`, `S3_CACHE_REGION` - Cache configuration

## Sample Workflow Usage

See `samples/` directory for example workflow files. Spring Boot apps typically use three workflow files:
- `build.yml` - Build and deploy to dev (triggers on branch push, ignores `v*` tags)
- `promote.yml` - Deploy to prod (triggers on `v*` tags)
- `release.yml` - Create Maven releases (manual `workflow_dispatch`)

For Java libraries:
- `ci-cd-lib.yml` - Build, test, and publish to Maven repository

## Runner Requirements

Workflows use self-hosted runners (`arc-runners-javadev`) with:
- Docker
- Java 21 (configurable via `java_version` input)
- Maven
- yq, jq, kubectl
- GitHub CLI (gh)

## Architecture Notes

- All workflows use `workflow_call` trigger for reusability
- Maven cache is stored in S3-compatible storage using `krystof-io/cache` action
- Docker images go to a private registry specified by `IMAGE_REGISTRY_HOST`
- GitOps updates use PR-based flow with auto-merge
- Deployment validation connects to Kubernetes clusters using service account tokens
