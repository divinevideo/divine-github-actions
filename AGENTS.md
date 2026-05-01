# Repository Guidelines

## Project Structure & Module Organization
- Reusable actions live in top-level directories such as `generate-docker-tags/` and `docker-push-gcp/`, each with its own `action.yml`.
- Shared documentation lives in `README.md`.
- Keep each action self-contained. If a new action is added, give it its own directory with focused inputs, outputs, and examples.

## Build, Test, and Development Commands
- There is no single build pipeline in this repo. Validate changes by reviewing `action.yml` metadata carefully and testing the action from a consuming workflow or a focused sandbox repo.
- If a change updates shell logic inside a composite action, run the relevant commands locally where practical before opening the PR.
- Keep examples in `README.md` aligned with the actual action inputs and outputs.

## Coding Style & Naming Conventions
- Keep action inputs, outputs, and descriptions explicit and stable.
- Prefer small, readable shell blocks inside composite actions over dense one-liners.
- Keep PRs tightly scoped. Do not mix unrelated action changes, doc rewrites, or release-tag work into the same PR.
- Temporary or transitional code must include `TODO(#issue):` with the tracking issue for removal.

## Pull Request Guardrails
- PR titles must use Conventional Commit format: `type(scope): summary` or `type: summary`.
- Set the correct PR title when opening the PR. Do not rely on fixing it afterward.
- If a PR title changes after opening, verify that the semantic PR title check reruns successfully.
- PR descriptions must include a short summary, motivation, linked issue, and manual test plan.
- Changes that affect action interfaces or examples should call out any required consumer follow-up clearly in the PR description.

## Security & Sensitive Information
- Do not commit secrets, live credentials, or private infrastructure identifiers in action examples or workflow snippets.
- Public issues, PRs, branch names, screenshots, and descriptions must not mention corporate partners, customers, brands, campaign names, or other sensitive external identities unless a maintainer explicitly approves it. Use generic descriptors instead.
