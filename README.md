# Divine GitHub Actions

Reusable GitHub Actions and workflows shared across Divine repositories. Each
action lives in its own top-level directory with an `action.yml`; the reusable
workflows live under `.github/workflows/`. Consuming repos reference them by
path with `uses:` so common CI plumbing — Docker tagging, GCP pushes, PR
auto-rebase, and agent-driven test fixes — lives in one place instead of being
copied into every repo.

## Available actions

Composite actions, each self-contained in its own directory.

### `generate-docker-tags`

Generates consistent Docker image tags from the current git context.

**Input:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `include-latest` | no | `true` | Add a `latest` tag on the `main` branch |

**Output:**

- `tags`: comma-separated list of tags (e.g. `main,abc1234,latest`)

**Tag logic:**

- Branch push: `{branch},{sha-short}` (plus `latest` when the branch is `main` and `include-latest` is `true`)
- Pull request: `pr-{number},{sha-short}`
- Tag push: `{tag},{sha-short}`
- Anything else: `{sha-short}`

The short SHA is the first 7 characters of `GITHUB_SHA`.

### `docker-push-gcp`

Authenticates to GCP via Workload Identity Federation and pushes a locally
built Docker image to Artifact Registry under each of the supplied tags.

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image-name` | yes | - | Local Docker image name with tag (e.g. `myapp:local`) |
| `registry-image` | yes | - | Target image name in the registry (e.g. `myapp`) |
| `tags` | yes | - | Comma-separated tags to push |
| `wif-provider` | yes | - | Workload Identity Federation provider URL |
| `service-account` | yes | - | GCP service account email |
| `project-id` | yes | - | GCP project ID |
| `repository` | yes | - | Artifact Registry repository name |
| `region` | no | `us-central1` | GCP region |

The calling job must grant the OIDC token permissions used for WIF:

```yaml
permissions:
  id-token: write
  contents: read
```

## Available reusable workflows

Two `workflow_call` workflows that ride on top of the divine-brain pipeline.
Each is **opt-in per repo** (one small caller workflow) and **opt-out per
branch** (drop a `.brain-autofix.disabled` file at the repo root, or label a PR
`do-not-touch` or `wip`).

Every brain-autofix commit carries both `[brain-autofix]` and `[skip-autofix]`
in its message. The `[skip-autofix]` trailer is the loop-break: when a workflow
sees it on the last commit, it exits so it never reacts to its own pushes.

### `auto-rebase` — keep PRs in sync with their base

Updates a PR branch when it falls behind base, either by merging base in
(`mode: merge`, the default) or rebasing and force-pushing with
`--force-with-lease` (`mode: rebase`). Trivial conflicts are resolved
automatically; anything else aborts cleanly and posts a PR comment listing the
files that need a human.

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `mode` | no | `merge` | `merge` (safe) or `rebase` (clean history, force-pushes) |
| `base-branch` | no | *(PR base, else `main`)* | Base branch to update from |

Trivial conflict resolution covers lockfiles — `pnpm-lock.yaml`,
`package-lock.json`, `yarn.lock`, `Cargo.lock`, `go.sum` (regenerated via the
relevant package manager when it is on the runner) — and generated artifacts in
`dist/`, `build/`, `.next/`, `.turbo/`, `node_modules/` (PR side kept, rebuilt
on the next CI run).

Only `pull_request` events do work; the job also runs its own guardrail checks
(labels, opt-out file, loop-break trailer, idempotency by base SHA) and never
pushes to the default branch. Fork PRs are skipped, because the base repo's
`GITHUB_TOKEN` can't push back to a fork branch.

### `auto-fix-tests` — agent fixes failing CI

When CI fails on an opted-in PR (label `auto-fix`, or a maintainer comments
`/brain-fix`), spins up Claude Code in the runner with a checkout of the PR
head, hands it the truncated failure log, and lets it produce one focused fix.
Tests are re-run to verify; the fix is committed and pushed only if they pass.

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `test-command` | yes | - | How to run tests (e.g. `pnpm test`, `cargo test`, `go test ./...`) |
| `install-command` | no | *(none)* | How to install deps before tests (e.g. `pnpm install --frozen-lockfile`) |
| `max-iterations` | no | `6` | Max agent read/edit/run-tests cycles |
| `anthropic-model` | no | `claude-opus-4-7` | Anthropic model id for the agent |

The caller repo must provide `ANTHROPIC_API_KEY` as a secret (or inherit it
from the org). It flows through `secrets:` and is never echoed. Fork PRs are
skipped for the same push-back reason as `auto-rebase`, and the workflow never
pushes to the default branch.

## Usage

Reference actions and workflows by their path within this repo:

- Actions are published at `@v1` and `@main`.
- The reusable workflows were added after `v1`, so pin them to `@main`.

### Docker build and push

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

      - name: Push to Artifact Registry
        if: github.event_name != 'pull_request'
        uses: divinevideo/divine-github-actions/docker-push-gcp@v1
        with:
          image-name: myapp:local
          registry-image: myapp
          tags: ${{ steps.tags.outputs.tags }}
          wif-provider: ${{ vars.WORKLOAD_IDENTITY_PROVIDER }}
          service-account: ${{ vars.SERVICE_ACCOUNT }}
          project-id: ${{ vars.GCP_PROJECT_ID }}
          repository: containers-poc
```

### Auto-rebase PRs

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
      mode: merge          # "merge" (default, safe) or "rebase" (force-with-lease)
    secrets: inherit
```

### Auto-fix failing tests

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
    secrets: inherit
```

## Development

There is no build pipeline. Validate changes by reviewing the `action.yml`
metadata carefully and exercising the action from a consuming workflow or a
focused sandbox repo. When a change updates shell logic inside a composite
action, run the affected commands locally where practical before opening the
PR, and keep the examples in this README aligned with the actual inputs and
outputs.

Conventions (see `AGENTS.md` for the full guidelines):

- Each action is self-contained in its own top-level directory with focused inputs, outputs, and examples.
- PR titles use Conventional Commit format (`type(scope): summary`); a Semantic PR check enforces this.
- Keep PRs tightly scoped — don't mix unrelated action changes, doc rewrites, or release-tag work.

## License

MIT

---

Part of [Divine](https://divine.video) — your playground for human creativity · [Brand guidelines](https://github.com/divinevideo/brand-guidelines)
