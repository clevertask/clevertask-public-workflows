# clevertask-public-workflows

Reusable GitHub Actions workflows consumed by CleverTask public repositories.

This repository is intentionally small in scope. It exists only to hold shared
automation for public/open-source repos that we want to maintain in one place.

## Available Workflows

- [`create-release-version.yml`](./.github/workflows/create-release-version.yml)
- [`publish-npm.yml`](./.github/workflows/publish-npm.yml)

## Design Notes

- Consumer repositories should keep thin local wrapper workflows.
- Local wrappers own the trigger (`workflow_dispatch`, `release`, etc.).
- Local wrappers map repository-specific secrets into the reusable workflows.
- Local wrappers keep stable workflow filenames for npm trusted publishing and
  repo-specific conventions.
- npm package builds and lifecycle checks run without an OIDC token. A fresh
  publish job verifies one packed tarball and publishes it with lifecycle
  scripts disabled.
- Release versioning disables package scripts and Git hooks, permits only the
  package manifest and optional lockfile to change, and receives the PAT only
  for remote release writes. Release and publish workflows support SemVer
  prereleases but reject build metadata that pnpm would remove.
- Shared workflows install exact `corepack@0.34.5`. Publish and release
  workflows select the exact pnpm version declared by `packageManager`.
- Dependabot owns the `github-actions` ecosystem in this repository. Consumer
  repositories use the same wiring so third-party Actions and immutable
  reusable-workflow references stay current without handing npm manifests or
  pnpm lockfiles to Dependabot.

## Dependency Maintenance

npm dependency updates are prepared and reviewed in consumer repositories. Each
update must keep Node.js and `@types/node` aligned and pass the repository's
compatibility checks and CI. Reviews can also evaluate technologies or APIs that
simplify the code and reduce maintenance work.

## Current Consumers

- `clevertask/kanban-board-ui`
- `clevertask/react-sortable-tree`
- `clevertask/scribe`
