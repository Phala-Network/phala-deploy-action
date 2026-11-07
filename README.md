# Phala Cloud CVM Deployment Action

Deploy CVMs to Phala Cloud using `phala.toml` configuration.

## Prerequisites

Your repository must contain a `phala.toml` file. All CVM configuration (name, compose file, instance type, region, privacy settings, etc.) is defined there. See [phala.toml documentation](https://docs.phala.network/) for the full schema.

For self-hosted runners, use Actions Runner `2.327.1` or newer: `setup-bun@v2.2.0` and the checkout version shown below use Node.js 24. GitHub-hosted `ubuntu-latest` runners support this runtime.

## Deployment completion

The action installs Phala CLI `1.2.2` by default and runs `phala deploy --json --wait`.

- For updates, it waits for the CVM to return to its pre-update power state, then checks the update operation outcome. A stopped CVM stays stopped.
- A failed or cancelled update causes the action to fail, including with keyless OIDC authentication. Success no longer means only that the update request was accepted.
- If nothing changed, deployment succeeds immediately without restarting the CVM.
- Updates requiring on-chain approval still return `status=pending_approval`; they are not reported as completed deployments.

If you override `cli-version`, use `1.2.1` or newer to retain OIDC outcome checks and correct waiting on stopped CVMs.

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
      - uses: actions/checkout@v7.0.1
      - uses: Phala-Network/phala-deploy-action@main
        with:
          api-key: ${{ secrets.PHALA_API_KEY }}
```

### Keyless with OIDC

No API key needed. Requires:

1. A **trusted repository** registered in your Phala Cloud workspace (admin/owner).
2. Workflow permissions `id-token: write` for OIDC and `contents: read` for checkout.
3. Phala CLI `>=1.2.1` (`1.2.2` by default in this action) for OIDC authentication and update outcome checks.

The action requests a GitHub OIDC token with audience `https://cloud.phala.com` (override via `oidc-audience` if your Cloud `APP_URL` differs) and sets `PHALA_OIDC_TOKEN` for the CLI. When waiting for an update, the CLI reads its operation outcome using the same token and a correlation ID scoped to the deployment target; no API key is needed.

```yaml
name: Deploy to Phala Cloud
on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7.0.1
      - uses: Phala-Network/phala-deploy-action@main
        # with:
          # Optional: required when this GitHub repo is trusted in multiple workspaces
          # workspace: my-workspace-slug
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
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7.0.1
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
| `workspace` | Workspace slug (`X-Phala-Workspace`). Use when the repo is trusted in multiple workspaces | No | |
| `oidc-audience` | OIDC audience (must match Cloud `APP_URL`) | No | `https://cloud.phala.com` |
| `cli-version` | Phala CLI version (use `>=1.2.1` for OIDC outcome checks and stopped-CVM waiting) | No | `1.2.2` |

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
| `oidc-repository` | Source repository when API returns `oidc_provenance` (may be empty) |
| `oidc-commit-sha` | Source commit SHA when API returns `oidc_provenance` (may be empty) |
| `commit-token` | Commit token for 2-step deployment |

## License

MIT
