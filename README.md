# Build and Push Docker Image to ECR

A composite GitHub Action that builds multi-platform Docker images and pushes them to Amazon ECR.
This will be used in the process of deploying to kubernetes

## Features

- Multi-platform Docker image builds (amd64/arm64)
- Automatic ECR authentication
- Docker layer caching for faster builds
- Optional Composer dependency installation
- Flexible composer authentication (workflow or Dockerfile)

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `aws-account` | AWS account name to assume role for | Yes | - |
| `platform` | Platform to build for (e.g., `linux/amd64` or `linux/arm64`) | Yes | - |
| `image-tag` | Image tag to use (e.g., `dev-123`) | Yes | - |
| `ecr-registry` | ECR registry URL | Yes | - |
| `github-sha` | GitHub commit SHA for tagging | Yes | - |
| `composer-auth` | Composer authentication token for GitHub | No | - |
| `install-dependencies` | Install composer dependencies in workflow before building | No | `''` |

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

### With Composer Auth as Build Arg (for Dockerfile)

When `install-dependencies` is not set, composer-auth is automatically passed as a build arg:

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
```

Your Dockerfile can then use:

```dockerfile
ARG COMPOSER_AUTH
RUN composer install --no-dev --optimize-autoloader
```

## How It Works

1. Checks out the repository
2. Assumes the specified AWS role for ECR access
3. (Optional) Installs Composer dependencies if `install-dependencies` is set
4. Sets up Docker Buildx for multi-platform builds
5. Logs into Amazon ECR
6. Builds and pushes the Docker image with:
   - Two tags: `{image-tag}-{arch}` and `{github-sha}-{arch}`
   - Layer caching for faster subsequent builds
   - Optional composer auth (as build arg if not installing in workflow)

## Image Tags

The action creates two tags for each build:

- `{image-tag}-{arch}` - e.g., `dev-123-amd64`
- `{github-sha}-{arch}` - e.g., `abc123def-arm64`

These architecture-specific tags can then be combined into a multi-platform manifest.


