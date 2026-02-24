# Build and Push Docker Image to ECR and Harbor

A composite GitHub Action that builds multi-platform Docker images and pushes them to both Amazon ECR and Harbor. Registry URLs are automatically constructed from the service name and namespace, so you don't need to pass them in. This action is used in the process of deploying to Kubernetes.

## Features

- Multi-platform Docker image builds (amd64/arm64)
- Automatic push to both Amazon ECR and Harbor
- Automatic ECR authentication
- Automatic registry URL construction (no need to pass ECR or Harbor URLs)
- Docker layer caching to Harbor for faster builds
- Optional Composer dependency installation in workflow
- Optional asset building (npm) before Docker build
- Flexible authentication options:
  - Create auth.json for Dockerfile consumption
  - GitHub Composer OAuth tokens
  - Flux UI credentials
  - Custom build arguments

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `platform` | Platform to build for (e.g., `linux/amd64` or `linux/arm64`) | Yes | - |
| `image-tag` | Image tag to use (e.g., `dev-123`) | Yes | - |
| `github-sha` | GitHub commit SHA for tagging | Yes | - |
| `service-name` | Name of service for location to store images in container repo | Yes | - |
| `namespace` | Namespace of app for Harbor | Yes | - |
| `aws-account` | AWS account name to assume role for | No | `bisnow` |
| `auth-json` | Create auth.json for composer/flux auth (set to any non-empty value to enable) | No | `''` |
| `composer-auth` | Composer authentication token for GitHub | No | - |
| `flux-username` | Flux UI username for http-basic authentication | No | - |
| `flux-license-key` | Flux UI license key for http-basic authentication | No | - |
| `php-version` | PHP version to set up (used when `auth-json` is enabled) | No | `8.3` |
| `install-dependencies` | Install composer dependencies in workflow before building | No | `''` |
| `build-assets` | Build npm assets before Docker build (set to any non-empty value to enable) | No | `''` |
| `no-cache` | Build without using cache (slower but ensures fresh build) | No | `false` |
| `build-args` | Custom build arguments to pass to Docker build | No | - |

## Registry URLs

The action automatically constructs registry URLs based on your inputs:

- **ECR**: `560285300220.dkr.ecr.us-east-1.amazonaws.com/{service-name}`
- **Harbor**: `harbor.bisnow.cloud/{namespace}/{service-name}`

For example, with `service-name: hello-k8s` and `namespace: bisnow`:
- **ECR**: `560285300220.dkr.ecr.us-east-1.amazonaws.com/hello-k8s`
- **Harbor**: `harbor.bisnow.cloud/bisnow/hello-k8s`

You don't need to pass these URLs - they're built automatically from `service-name` and `namespace`.

## Usage

### Basic Usage

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build and Push Image
        uses: bisnow/github-actions-build-push-k8s@main
        with:
          platform: linux/amd64
          image-tag: dev-${{ github.run_number }}
          github-sha: ${{ github.sha }}
          service-name: hello-k8s
          namespace: bisnow
```

### Multi-Platform Build

```yaml
jobs:
  build-and-push:
    runs-on: ${{ matrix.runner }}
    strategy:
      matrix:
        include:
          - platform: linux/amd64
            runner: ubuntu-latest-amd64
          - platform: linux/arm64
            runner: ubuntu-latest-arm64
    steps:
      - name: Build and Push ${{ matrix.platform }} Image
        uses: bisnow/github-actions-build-push-k8s@main
        with:
          platform: ${{ matrix.platform }}
          image-tag: dev-${{ github.run_number }}
          github-sha: ${{ github.sha }}
          service-name: hello-k8s
          namespace: bisnow
          composer-auth: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

### With Composer Dependencies Installed in Workflow

This is useful when you want dependencies in the Docker build context (cached):

```yaml
steps:
  - name: Build and Push Image
    uses: bisnow/github-actions-build-push-k8s@main
    with:
      platform: linux/amd64
      image-tag: dev-${{ github.run_number }}
      github-sha: ${{ github.sha }}
      service-name: hello-k8s
      namespace: bisnow
      composer-auth: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
      install-dependencies: 'true'
```

### With Asset Building

Build npm assets before the Docker build. This sets up Node.js 22.x and runs `npm ci` followed by `npm run build`:

```yaml
steps:
  - name: Build and Push Image
    uses: bisnow/github-actions-build-push-k8s@main
    with:
      platform: linux/amd64
      image-tag: dev-${{ github.run_number }}
      github-sha: ${{ github.sha }}
      service-name: hello-k8s
      namespace: bisnow
      build-assets: 'true'
```

The built assets will be included in the Docker build context, allowing you to copy them into your image without running npm during the Docker build.

### With auth.json for Dockerfile (Recommended)

Creates an auth.json file that your Dockerfile can copy and use. This approach supports both GitHub and Flux credentials:

```yaml
steps:
  - name: Build and Push Image
    uses: bisnow/github-actions-build-push-k8s@main
    with:
      aws-account: bisnow
      platform: linux/amd64
      image-tag: dev-${{ github.run_number }}
      github-sha: ${{ github.sha }}
      service-name: hello-k8s
      namespace: bisnow
      auth-json: 'true'
      composer-auth: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
      flux-username: ${{ secrets.FLUX_USERNAME }}
      flux-license-key: ${{ secrets.FLUX_LICENSE_KEY }}
```

Your Dockerfile can then use:

```dockerfile
# Copy all files including auth.json created by GitHub Actions
COPY --chown=www-data:www-data ./ /app/

# Install dependencies using auth.json for authentication, then remove auth.json for security
RUN composer install \
    --no-dev \
    --no-scripts \
    --no-interaction \
    --prefer-dist \
    --optimize-autoloader \
    && composer run-script post-autoload-dump --no-interaction \
    && rm -f auth.json
```

### With Custom Build Arguments

Pass custom build arguments to your Docker build:

```yaml
steps:
  - name: Build and Push Image
    uses: bisnow/github-actions-build-push-k8s@main
    with:
      platform: linux/amd64
      image-tag: dev-${{ github.run_number }}
      github-sha: ${{ github.sha }}
      service-name: hello-k8s
      namespace: bisnow
      build-args: |
        APP_ENV=production
        COMPOSER_AUTH=${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

### Force Rebuild Without Cache

Useful for troubleshooting or ensuring a completely fresh build:

```yaml
steps:
  - name: Build and Push Image
    uses: bisnow/github-actions-build-push-k8s@main
    with:
      aws-account: bisnow
      platform: linux/amd64
      image-tag: dev-${{ github.run_number }}
      github-sha: ${{ github.sha }}
      service-name: hello-k8s
      namespace: bisnow
      no-cache: true
```

## How It Works

1. Validates that `service-name` and `namespace` are provided
2. Constructs registry URLs automatically:
   - ECR: `560285300220.dkr.ecr.us-east-1.amazonaws.com/{service-name}`
   - Harbor: `harbor.bisnow.cloud/{namespace}/{service-name}`
3. Checks out the repository
4. (Optional) Sets up Node.js 22.x and builds assets if `build-assets` is enabled
   - Runs `npm ci` to install dependencies
   - Runs `npm run build` to build assets
5. (Optional) Sets up PHP and configures Composer credentials if `auth-json` is enabled
   - Creates auth.json file with GitHub OAuth and/or Flux credentials
   - Supports both GitHub and Flux UI authentication
6. Assumes the specified AWS role for ECR access
7. (Optional) Installs Composer dependencies if `install-dependencies` is set
8. Sets up Docker Buildx for multi-platform builds
9. Logs into Amazon ECR (Harbor authentication is handled by the runner)
10. Builds and pushes the Docker image to both registries with:
    - Four tags total (two per registry): `{image-tag}-{arch}` and `{github-sha}-{arch}`
    - Registry layer caching to Harbor for faster subsequent builds
    - Optional custom build arguments

## Image Tags

The action creates four tags for each build (two per registry):

**ECR Tags:**
- `{image-tag}-{arch}` - e.g., `dev-123-amd64`
- `{github-sha}-{arch}` - e.g., `abc123def-amd64`

**Harbor Tags:**
- `{image-tag}-{arch}` - e.g., `dev-123-amd64`
- `{github-sha}-{arch}` - e.g., `abc123def-amd64`

These architecture-specific tags can then be combined into a multi-platform manifest.

## Caching

The action uses Harbor for Docker layer caching:
- **Cache source**: `harbor.bisnow.cloud/{namespace}/{service-name}:cache-{arch}`
- **Cache destination**: `harbor.bisnow.cloud/{namespace}/{service-name}:cache-{arch}`

This ensures faster builds by reusing cached layers from previous builds stored in Harbor.

## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.
