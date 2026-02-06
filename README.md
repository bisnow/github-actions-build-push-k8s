# Build and Push Docker Image to ECR

A composite GitHub Action that builds multi-platform Docker images and pushes them to Amazon ECR.
This will be used in the process of deploying to kubernetes

## Features

- Multi-platform Docker image builds (amd64/arm64)
- Automatic ECR authentication
- Docker layer caching for faster builds
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
| `aws-account` | AWS account name to assume role for | Yes | - |
| `platform` | Platform to build for (e.g., `linux/amd64` or `linux/arm64`) | Yes | - |
| `image-tag` | Image tag to use (e.g., `dev-123`) | Yes | - |
| `ecr-registry` | ECR registry URL | Yes | - |
| `github-sha` | GitHub commit SHA for tagging | Yes | - |
| `auth-json` | Create auth.json for composer/flux auth (set to any non-empty value to enable) | No | `''` |
| `composer-auth` | Composer authentication token for GitHub | No | - |
| `flux-username` | Flux UI username for http-basic authentication | No | - |
| `flux-license-key` | Flux UI license key for http-basic authentication | No | - |
| `php-version` | PHP version to set up (used when `auth-json` is enabled) | No | `8.3` |
| `install-dependencies` | Install composer dependencies in workflow before building | No | `''` |
| `build-assets` | Build npm assets before Docker build (set to any non-empty value to enable) | No | `''` |
| `no-cache` | Build without using cache (slower but ensures fresh build) | No | `false` |
| `build-args` | Custom build arguments to pass to Docker build | No | - |

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
          aws-account: bisnow
          platform: linux/amd64
          image-tag: dev-${{ github.run_number }}
          ecr-registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/myapp
          github-sha: ${{ github.sha }}
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
          aws-account: bisnow
          platform: ${{ matrix.platform }}
          image-tag: dev-${{ github.run_number }}
          ecr-registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/myapp
          github-sha: ${{ github.sha }}
          composer-auth: ${{ secrets.COMPOSER_OAUTH_GITHUB_ACTIONS }}
```

### With Composer Dependencies Installed in Workflow

This is useful when you want dependencies in the Docker build context (cached):

```yaml
steps:
  - name: Build and Push Image
    uses: bisnow/github-actions-build-push-k8s@main
    with:
      aws-account: bisnow
      platform: linux/amd64
      image-tag: dev-${{ github.run_number }}
      ecr-registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/myapp
      github-sha: ${{ github.sha }}
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
      aws-account: bisnow
      platform: linux/amd64
      image-tag: dev-${{ github.run_number }}
      ecr-registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/myapp
      github-sha: ${{ github.sha }}
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
      ecr-registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/myapp
      github-sha: ${{ github.sha }}
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
      aws-account: bisnow
      platform: linux/amd64
      image-tag: dev-${{ github.run_number }}
      ecr-registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/myapp
      github-sha: ${{ github.sha }}
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
      ecr-registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/myapp
      github-sha: ${{ github.sha }}
      no-cache: true
```

## How It Works

1. Checks out the repository
2. (Optional) Sets up Node.js 22.x and builds assets if `build-assets` is enabled
   - Runs `npm ci` to install dependencies
   - Runs `npm run build` to build assets
3. (Optional) Sets up PHP and configures Composer credentials if `auth-json` is enabled
   - Creates auth.json file with GitHub OAuth and/or Flux credentials
   - Supports both GitHub and Flux UI authentication
4. Assumes the specified AWS role for ECR access
5. (Optional) Installs Composer dependencies if `install-dependencies` is set
6. Sets up Docker Buildx for multi-platform builds
7. Logs into Amazon ECR
8. Builds and pushes the Docker image with:
   - Two tags: `{image-tag}-{arch}` and `{github-sha}-{arch}`
   - Registry layer caching for faster subsequent builds
   - Optional custom build arguments

## Image Tags

The action creates two tags for each build:

- `{image-tag}-{arch}` - e.g., `dev-123-amd64`
- `{github-sha}-{arch}` - e.g., `abc123def-arm64`

These architecture-specific tags can then be combined into a multi-platform manifest.

## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.
