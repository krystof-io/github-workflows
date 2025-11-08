# CI/CD Pipeline Setup Guide

Complete setup instructions for the GitHub Actions CI/CD pipeline with Flux GitOps deployment.

## Architecture Overview

This CI/CD pipeline consists of:
- **Shared Workflows Repository**: Reusable GitHub Actions workflows
- **Docker Templates Repository**: Generic Dockerfiles for applications
- **Application Repositories**: Individual repos for each app/library
- **GitOps Repositories**: Flux-managed Kubernetes manifests (one per cluster)

## Prerequisites

- GitHub organization with repositories created
- Two Kubernetes clusters with Flux installed
- Private Docker registry (Nexus)
- Private Maven repository (Nexus)
- SonarQube instance (optional)
- GitHub self-hosted runners

## Repository Setup

### 1. Create Required Repositories

Create the following repositories in your GitHub organization:

```bash
# Infrastructure repositories
github-workflows          # Shared reusable workflows
docker-templates         # Generic Dockerfiles
build-cluster-gitops     # GitOps manifests for build cluster
app-cluster-gitops       # GitOps manifests for app cluster

# Application/Library repositories (examples)
my-service              # Your Spring Boot application
my-library              # Your Java library
```

### 2. Set Up Shared Workflows Repository

**Repository**: `github-workflows`

```bash
git clone git@github.com:krystof-io/github-workflows.git
cd github-workflows

# Create directory structure
mkdir -p .github/workflows config

# Add files (from artifacts provided)
# - config/clusters.yml
# - .github/workflows/java-app-build.yml
# - .github/workflows/java-lib-build.yml

git add .
git commit -m "Initial shared workflows setup"
git push origin main
```

**Edit `config/clusters.yml`** and update with your actual repository names:
```yaml
clusters:
  build:
    gitops_repo: 'krystof-io/build-cluster-gitops'
    cluster_path: 'build-cluster'
  app:
    gitops_repo: 'krystof-io/app-cluster-gitops'
    cluster_path: 'app-cluster'
```

### 3. Set Up Docker Templates Repository

**Repository**: `docker-templates`

```bash
git clone git@github.com:krystof-io/docker-templates.git
cd docker-templates

# Create directory structure
mkdir -p java/spring-boot node

# Add files (from artifacts provided)
# - java/spring-boot/Dockerfile

git add .
git commit -m "Initial Docker templates"
git push origin main
```

### 4. Set Up GitOps Repositories

**For both**: `build-cluster-gitops` and `app-cluster-gitops`

```bash
git clone git@github.com:krystof-io/build-cluster-gitops.git  # or app-cluster-gitops
cd build-cluster-gitops

# Create directory structure (example for one app)
mkdir -p .github/workflows
mkdir -p clusters/build-cluster/namespaces/kio-dev/my-service
mkdir -p clusters/build-cluster/namespaces/kio-prod/my-service

# Add validation workflow
# - .github/workflows/validate-pr.yml

# Add application manifests (HelmRelease or Kustomization)
# Example: clusters/build-cluster/namespaces/kio-dev/my-service/helmrelease.yaml

git add .
git commit -m "Initial GitOps structure"
git push origin main
```

### 5. Set Up Application Repository

**Repository**: `my-service` (example)

```bash
git clone git@github.com:krystof-io/my-service.git
cd my-service

# Create workflow directory
mkdir -p .github/workflows

# Add files (from artifacts provided)
# - .github/workflows/ci-cd.yml
# - pom.xml

# Update ci-cd.yml with your org name
sed -i 's/krystof-io/YOUR_ACTUAL_ORG/g' .github/workflows/ci-cd.yml

git add .
git commit -m "Add CI/CD workflow"
git push origin main
```

## GitHub Secrets Configuration

### Organization-Level Secrets (Recommended)

Configure these secrets at the organization level so all repositories can access them:

```
Settings → Secrets and variables → Actions → New organization secret
```

**Required Secrets:**

| Secret Name | Description | Example Value |
|------------|-------------|---------------|
| `IMAGE_REGISTRY_USERNAME` | Docker registry username | `cicd-user` |
| `IMAGE_REGISTRY_PASSWORD` | Docker registry password | `***` |
| `MAVEN_REPO_USERNAME` | Maven repository username | `cicd-user` |
| `MAVEN_REPO_PASSWORD` | Maven repository password | `***` |
| `CICD_TOKEN` | GitHub Personal Access Token with `repo` scope | `ghp_***` |
| `CICD_USER_NAME` | Git commit author name | `github-actions[bot]` |
| `CICD_USER_EMAIL` | Git commit author email | `github-actions[bot]@users.noreply.github.com` |
| `SONAR_TOKEN` | SonarQube authentication token (optional) | `squ_***` |


**Notes:**
- The `CICD_TOKEN` must have write access to GitOps repositories and PR creation permissions
- For GitOps repos, also add `IMAGE_REGISTRY_USERNAME` and `IMAGE_REGISTRY_PASSWORD` as repository secrets
- All secrets can be configured at the organization level for easier management

### Organization-Level Environment Variables

Configure these environment variables at the organization level:

```
Settings → Secrets and variables → Actions → Variables → New organization variable
```

**Required Variables:**

| Variable Name | Description | Value |
|--------------|-------------|-------|
| `IMAGE_REGISTRY_HOST` | Docker registry hostname | `docker-private.build.krystof.io` |
| `MAVEN_REPO_URL` | Maven public repository URL | `http://registry.build.krystof.io/repository/maven-public` |
| `MAVEN_PRIVATE_RELEASE_REPO_URL` | Maven releases repository URL | `https://registry.build.krystof.io/repository/maven-releases/` |
| `MAVEN_PRIVATE_SNAPSHOT_REPO_URL` | Maven snapshots repository URL | `https://registry.build.krystof.io/repository/maven-snapshots/` |

These variables are automatically available to all workflows in the organization.

## Self-Hosted Runner Setup

### Install Required Tools

On each self-hosted runner machine:

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install Java (using SDKMAN)
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install java 17.0.9-tem
sdk default java 17.0.9-tem

# Install Maven
sdk install maven 3.9.5

# Install yq (YAML processor)
sudo wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/bin/yq
sudo chmod +x /usr/bin/yq

# Install kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/

# Install GitHub CLI
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh -y
```

### Configure Maven Settings

Create `~/.m2/settings.xml` on the runner:

```xml
<settings>
  <servers>
    <server>
      <id>maven-releases</id>
      <username>${env.MAVEN_REPO_USERNAME}</username>
      <password>${env.MAVEN_REPO_PASSWORD}</password>
    </server>
    <server>
      <id>maven-snapshots</id>
      <username>${env.MAVEN_REPO_USERNAME}</username>
      <password>${env.MAVEN_REPO_PASSWORD}</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>maven-public</id>
      <mirrorOf>*</mirrorOf>
      <url>${env.MAVEN_REPO_URL}</url>
    </mirror>
  </mirrors>
</settings>
```

**Note**: The environment variables (`MAVEN_REPO_URL`, `MAVEN_REPO_USERNAME`, `MAVEN_REPO_PASSWORD`) are injected by GitHub Actions from the organization-level configuration.

### Register GitHub Runner

```bash
# Navigate to your organization's settings
# Settings → Actions → Runners → New self-hosted runner

# Follow the instructions to download and configure the runner
# Add the 'self-hosted' label during setup
```

## Usage

### Creating a New Application

1. **Create repository** from template or start fresh
2. **Add pom.xml** with your application configuration
3. **Add CI/CD workflow file** (`.github/workflows/ci-cd.yml`):
   ```yaml
   name: CI/CD Pipeline
   on:
     push:
       branches: [main, 'feature/**']
       tags: ['v*']
   jobs:
     build-and-deploy:
       uses: krystof-io/github-workflows/.github/workflows/java-app-build.yml@main
       with:
         app_name: my-new-service
       secrets: inherit
   ```
4. **Add Maven release workflow file** (`.github/workflows/release.yml`):
   ```yaml
   name: Maven Release
   on:
     workflow_dispatch:
       inputs:
         release_version:
           description: 'Release version (e.g., 1.0.0)'
           required: true
           type: string
         next_development_version:
           description: 'Next development version (e.g., 1.1.0-SNAPSHOT)'
           required: true
           type: string
         dry_run:
           description: 'Perform a dry run (no actual release)'
           required: false
           type: boolean
           default: false
   jobs:
     maven-release:
       uses: krystof-io/github-workflows/.github/workflows/maven-release.yml@main
       with:
         app_name: my-new-service
         release_version: ${{ github.event.inputs.release_version }}
         next_development_version: ${{ github.event.inputs.next_development_version }}
         dry_run: ${{ github.event.inputs.dry_run }}
       secrets: inherit
   ```
5. **Create GitOps manifests** in both cluster repos (if deploying to both)
6. **Push to main** - your app will build and deploy to dev!

### Creating a Release

There are two ways to create releases:

#### Option 1: GitHub Actions Workflow (Recommended)

1. **Navigate to Actions tab** in your repository
2. **Select "Maven Release" workflow**
3. **Click "Run workflow"**
4. **Fill in parameters**:
   - Release version: `1.0.0`
   - Next development version: `1.1.0-SNAPSHOT`
   - Dry run: Check for testing first
5. **Review the results**:
   - Creates git tag (e.g., `v1.0.0`)
   - Creates PR with version updates
   - Automatic production deployment triggered by tag

#### Option 2: Manual Maven Release (Traditional)

1. **Ensure main branch is stable** and all tests pass
2. **Run Maven release**:
   ```bash
   mvn release:prepare release:perform
   ```
3. **Maven release plugin will**:
   - Remove `-SNAPSHOT` from version
   - Create git tag (e.g., `v1.0.0`)
   - Push tag to GitHub
   - Bump version to next SNAPSHOT

#### What Happens After Release (Both Methods)

4. **Tag push triggers CI/CD workflow** which:
   - Builds Docker image with release version
   - Creates PR in GitOps repo(s) for production
5. **Review and merge PR** (or wait for auto-merge if checks pass)
6. **Flux deploys** to production automatically

#### GitHub Release Creation

The `github-release.yml` workflow automatically creates GitHub releases when tags are pushed, including:
- Release notes from git commits
- JAR file as downloadable artifact

### Branch Strategy

- **`feature/*`, `feat/*`, `bugfix/*`, `fix/*`**: Deploy to dev with feature-specific tags
- **`main`**: Deploy to dev with SNAPSHOT versions
- **Tags `v*`**: Create PR for production deployment

## Troubleshooting

### Build Failures

**Problem**: Maven build fails with "Cannot resolve dependencies"

**Solution**: 
- Check Maven settings.xml is configured correctly
- Verify `MAVEN_REPO_USERNAME` and `MAVEN_REPO_PASSWORD` secrets
- Ensure Nexus is accessible from runner

---

**Problem**: Docker build fails with "JAR file not found"

**Solution**:
- Ensure `mvn package` completes successfully
- Check the JAR is in the `target/` directory
- Verify the Spring Boot Maven plugin is configured in pom.xml

### Deployment Failures

**Problem**: GitOps update fails with "Permission denied"

**Solution**:
- Verify `CICD_TOKEN` has write access to GitOps repositories
- Check token hasn't expired
- Ensure token has `repo` scope

---

**Problem**: PR auto-merge doesn't work for production

**Solution**:
- Check validation workflow in GitOps repo is passing
- Verify auto-merge is enabled on the PR
- Look for failed checks in the PR

---

**Problem**: Image not found in registry

**Solution**:
- Check Docker build step completed successfully
- Verify `IMAGE_REGISTRY_USERNAME` and `IMAGE_REGISTRY_PASSWORD` are correct
- Ensure runner can push to the registry
- Check registry URL is correct

### Integration Test Failures

**Problem**: Testcontainers can't start containers

**Solution**:
- Ensure Docker is installed and running on the runner
- Verify the runner user has permissions to access Docker socket
- Check `/var/run/docker.sock` permissions

---

**Problem**: Integration tests only run sometimes

**Solution**:
- Integration tests only run on `main` branch by design
- Check the workflow log to confirm branch detection

## Workflow Customization

### Custom Dockerfile

If your app needs a custom Dockerfile, you can override:

```yaml
with:
  app_name: my-service
  dockerfile_repo: 'my-org/my-custom-dockerfiles'
  dockerfile_path: 'custom/MyDockerfile'
```

### Custom Java Version

```yaml
with:
  app_name: my-service
  java_version: '21'
```

### Deploy to Multiple Clusters

```yaml
with:
  app_name: my-service
  target_clusters: '["build", "app"]'
```

### Build Only (No Deployment)

If you want to build, test, and create Docker images without deploying anywhere, simply leave `target_clusters` empty:

```yaml
with:
  app_name: my-service
  target_clusters: '[]'  # Empty array - no deployment
```

This is useful for:
- Libraries that don't need deployment
- Applications in early development
- CI-only workflows for testing
- Building images for manual deployment

The workflow will:
- ✅ Build and test the application
- ✅ Run Sonar analysis (on main branch)
- ✅ Build and push Docker images  
- ❌ Skip all deployment steps

**Note**: Prior to this fix, leaving `target_clusters` empty would cause the workflow to fail with "matrix must define at least one vector". This has been resolved by making the deployment jobs conditional.

### Custom Heap Size

```yaml
with:
  app_name: my-service
  heap_size: '2g'
```

## Adding a New Cluster

1. **Update `config/clusters.yml`** in `github-workflows` repo:
   ```yaml
   staging:
     gitops_repo: 'krystof-io/staging-cluster-gitops'
     cluster_path: 'staging-cluster'
   ```

2. **Apps can now target it**:
   ```yaml
   with:
     target_clusters: '["staging"]'
   ```

No other changes needed!

## Monitoring and Observability

### GitHub Actions Dashboard

Monitor all workflows at:
```
https://github.com/krystof-io/my-service/actions
```

### GitOps Repository PRs

Production deployment PRs are visible at:
```
https://github.com/krystof-io/app-cluster-gitops/pulls
```

### Flux Status

Check Flux reconciliation status in your cluster:
```bash
flux get all -n flux-system
flux logs -n flux-system
```

## Best Practices

1. **Always test on feature branches first** before merging to main
2. **Use semantic versioning** for releases (major.minor.patch)
3. **Keep shared workflows up to date** across all applications
4. **Monitor failed builds** and fix promptly
5. **Review production PRs** even with auto-merge enabled
6. **Use meaningful commit messages** - they appear in GitOps commits
7. **Tag releases consistently** - always use `v` prefix (e.g., `v1.2.3`)

## Support

For issues or questions:
1. Check the workflow logs in GitHub Actions
2. Review this documentation
3. Check GitOps repository PR comments for validation errors
4. Contact your DevOps team

## License

Internal use only - Krystof.io