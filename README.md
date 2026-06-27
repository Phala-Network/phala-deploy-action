# Phala Cloud CVM Deployment Action

Deploy CVMs to Phala Cloud using `phala.toml` configuration.

## Prerequisites

Your repository must contain a `phala.toml` file. All CVM configuration (name, compose file, instance type, region, privacy settings, etc.) is defined there. See [phala.toml documentation](https://docs.phala.network/) for the full schema.

## Usage

### Basic (API Key)

```yaml
name: Deploy to Phala Cloud
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Phala-Network/phala-deploy-action@main
        with:
          api-key: ${{ secrets.PHALA_API_KEY }}
```

### Keyless with OIDC

No API key needed. Requires a [trusted repository](https://docs.phala.network/) configured in your Phala Cloud workspace.

```yaml
name: Deploy to Phala Cloud
on:
  push:
    branches: [main]

permissions:
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Phala-Network/phala-deploy-action@main
```

### Auto-deploy on config change

Trigger deployment when `phala.toml` or the compose file changes:

```yaml
name: Deploy to Phala Cloud
on:
  push:
    branches: [main]
    paths:
      - 'phala.toml'
      - 'docker-compose.yml'

permissions:
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Phala-Network/phala-deploy-action@main
        with:
          api-key: ${{ secrets.PHALA_API_KEY }}
```

### With environment variables from secrets

```yaml
      - uses: Phala-Network/phala-deploy-action@main
        with:
          api-key: ${{ secrets.PHALA_API_KEY }}
          envs: |
            DATABASE_URL=${{ secrets.DATABASE_URL }}
            REDIS_URL=${{ secrets.REDIS_URL }}
```

### On-chain KMS (2-step deployment)

For CVMs using on-chain KMS, the action exits successfully with `status=pending_approval` instead of failing. The compose hash must be approved on-chain before the update takes effect.

```yaml
      - uses: Phala-Network/phala-deploy-action@main
        id: deploy
        with:
          api-key: ${{ secrets.PHALA_API_KEY }}

      - name: Check deployment status
        if: steps.deploy.outputs.status == 'pending_approval'
        run: |
          echo "Deployment requires on-chain approval"
          echo "Compose hash: ${{ steps.deploy.outputs.compose-hash }}"
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `api-key` | Phala Cloud API Key. Optional when OIDC is enabled | No | |
| `envs` | Environment variables (KEY=VALUE per line) | No | |
| `cli-version` | Phala CLI version | No | `1.1.19` |

## Outputs

| Output | Description |
|--------|-------------|
| `result` | Full CVM info as JSON |
| `status` | `success`, `failed`, or `pending_approval` |
| `app-id` | App ID |
| `vm-uuid` | VM UUID |
| `cvm-name` | CVM name |
| `dashboard-url` | Dashboard URL |
| `compose-hash` | Normalized app-compose SHA256 |
| `docker-compose-hash` | Raw docker-compose.yml SHA256 |
| `pre-launch-script-hash` | Pre-launch script SHA256 |
| `os-image` | OS image name |
| `os-image-version` | OS image version |
| `os-image-hash` | OS image hash |
| `kms-type` | Key management type |
| `device-id` | Device ID |
| `oidc-repository` | Source repository (OIDC) |
| `oidc-commit-sha` | Source commit SHA (OIDC) |
| `commit-token` | Commit token for 2-step deployment |

## License

MIT
