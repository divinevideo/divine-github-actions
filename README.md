# Divine GitHub Actions

Reusable GitHub Actions for DiVine repositories.

## Actions

### `generate-docker-tags`

Generates consistent Docker image tags from git context.

```yaml
- name: Generate tags
  id: tags
  uses: divinevideo/divine-github-actions/generate-docker-tags@v1
  with:
    include-latest: 'true'  # Add 'latest' tag on main branch (default: true)

# Use the output
- run: echo "Tags: ${{ steps.tags.outputs.tags }}"
```

**Outputs:**
- `tags`: Comma-separated list of tags (e.g., `main,abc1234,latest`)

**Tag Logic:**
- Branch push: `{branch},{sha-short}` (+ `latest` if main)
- Pull request: `pr-{number},{sha-short}`

### `docker-push-gcp`

Authenticates to GCP and pushes a Docker image to Artifact Registry.

```yaml
- name: Push to POC
  uses: divinevideo/divine-github-actions/docker-push-gcp@v1
  with:
    image-name: myapp:local
    registry-image: myapp
    tags: main,abc1234,latest
    wif-provider: ${{ vars.WORKLOAD_IDENTITY_PROVIDER }}
    service-account: ${{ vars.SERVICE_ACCOUNT }}
    project-id: ${{ vars.GCP_PROJECT_ID }}
    repository: containers-poc
    region: us-central1  # optional, default: us-central1
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image-name` | yes | - | Local Docker image name with tag (e.g., `myapp:local`) |
| `registry-image` | yes | - | Target image name in registry (e.g., `myapp`) |
| `tags` | yes | - | Comma-separated tags to push |
| `wif-provider` | yes | - | Workload Identity Federation provider URL |
| `service-account` | yes | - | GCP service account email |
| `project-id` | yes | - | GCP project ID |
| `repository` | yes | - | Artifact Registry repository name |
| `region` | no | `us-central1` | GCP region |

**Required Permissions:**

The calling workflow must have:
```yaml
permissions:
  id-token: write
  contents: read
```

## Full Example

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Generate tags
        id: tags
        uses: divinevideo/divine-github-actions/generate-docker-tags@v1

      - name: Build Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          load: true
          tags: myapp:local

      - name: Push to POC
        if: github.event_name != 'pull_request'
        uses: divinevideo/divine-github-actions/docker-push-gcp@v1
        with:
          image-name: myapp:local
          registry-image: myapp
          tags: ${{ steps.tags.outputs.tags }}
          wif-provider: ${{ vars.WORKLOAD_IDENTITY_PROVIDER_POC }}
          service-account: ${{ vars.SERVICE_ACCOUNT_POC }}
          project-id: ${{ vars.GCP_PROJECT_ID_POC }}
          repository: containers-poc

      - name: Push to Production
        if: github.event_name != 'pull_request'
        uses: divinevideo/divine-github-actions/docker-push-gcp@v1
        with:
          image-name: myapp:local
          registry-image: myapp
          tags: ${{ steps.tags.outputs.tags }}
          wif-provider: ${{ vars.WORKLOAD_IDENTITY_PROVIDER_PROD }}
          service-account: ${{ vars.SERVICE_ACCOUNT_PROD }}
          project-id: ${{ vars.GCP_PROJECT_ID_PROD }}
          repository: containers-production
```

## License

MIT
