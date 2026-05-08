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

## Reusable workflows (brain-autofix)

Two reusable workflows that ride on top of the divine-brain pipeline. Each
one is designed to be **opt-in per repo** (one caller workflow, ~10 lines)
and **opt-out per branch** (drop a `.brain-autofix.disabled` file at the
repo root or label a PR `do-not-touch`).

All brain-autofix commits carry both `[brain-autofix]` and `[skip-autofix]`
in the message. The `[skip-autofix]` trailer is the loop-break — when the
workflow sees it on the last commit, it bails out so it never reacts to
its own pushes.

### `auto-rebase` — keep PRs in sync with main

Updates a PR branch when it falls behind base. Resolves trivial conflicts
automatically (lockfiles regenerate, generated artifacts kept PR-side);
posts a comment listing files needing human help on anything else.

```yaml
# .github/workflows/auto-update-pr.yml in the consumer repo
name: Auto-update PRs
on:
  pull_request:
    types: [synchronize, labeled, opened, reopened, ready_for_review]
permissions:
  contents: write
  pull-requests: write
jobs:
  auto-rebase:
    uses: divinevideo/divine-github-actions/.github/workflows/auto-rebase.yml@main
    with:
      mode: merge          # "merge" (safe, default) or "rebase" (force-with-lease)
    secrets: inherit
```

**Note on triggers:** v0 only acts on `pull_request` events. A `push: main` trigger that fanned out across all open PRs would be a useful follow-up but isn't built — don't add `push:` to your caller for now.

**Fork PRs are skipped** automatically. The workflow can't push back to a fork branch using the base repo's `GITHUB_TOKEN`.

**Trivial conflict resolution covers:**
`pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `Cargo.lock`, `go.sum`
(regenerated via the relevant package manager), and `dist/`, `build/`,
`.next/`, `.turbo/`, `node_modules/` (PR side wins; rebuilt on next CI).

### `auto-fix-tests` — agent fixes failing CI

When CI fails on a PR with the `auto-fix` label (or someone comments
`/brain-fix` on the PR), spins up Claude Code in the runner with a
checkout, hands it the failure log, and lets it produce one focused
fix commit. Re-runs tests once to verify; only pushes if they pass.

```yaml
# .github/workflows/auto-fix.yml in the consumer repo
name: Auto-fix tests
on:
  workflow_run:
    workflows: ["build-and-deploy"]   # name of your CI workflow
    types: [completed]
  pull_request:
    types: [labeled]
  issue_comment:
    types: [created]
permissions:
  contents: write
  pull-requests: write
  actions: read
jobs:
  fix:
    uses: divinevideo/divine-github-actions/.github/workflows/auto-fix-tests.yml@main
    with:
      test-command: 'pnpm test'
      install-command: 'pnpm install --frozen-lockfile'
      max-iterations: 6
      anthropic-model: 'claude-opus-4-7'
    secrets: inherit
```

The caller repo must have `ANTHROPIC_API_KEY` set as a secret (or
inherited from the org).

**Fork PRs are skipped** — `secrets.GITHUB_TOKEN` can't push to a fork.

### Guardrails (both workflows)

- **Opt-out per repo:** `.brain-autofix.disabled` file at repo root.
- **Opt-out per PR:** label `do-not-touch` or `wip`.
- **Loop-break:** every brain-autofix commit carries `[skip-autofix]`;
  the workflow skips when it sees that trailer on the last commit.
- **Idempotency:** `auto-fix-tests` skips re-fires for the same head_sha
  via a tagged comment lookup.
- **Never pushes to default branch:** explicit defense-in-depth check.
- **Force-push only with `--force-with-lease`:** never plain `--force`.
- **Secrets:** `ANTHROPIC_API_KEY` flows via `secrets:` not `env:`,
  never echoed.

## License

MIT
