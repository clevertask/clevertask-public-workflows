# clevertask-public-workflows

Reusable GitHub Actions workflows consumed by CleverTask public repositories.

This repository is intentionally small in scope. It exists only to hold shared
automation for public/open-source repos that we want to maintain in one place.

## Available Workflows

- [`create-release-version.yml`](./.github/workflows/create-release-version.yml)
- [`publish-npm.yml`](./.github/workflows/publish-npm.yml)
- [`update-pnpm-dependencies.yml`](./.github/workflows/update-pnpm-dependencies.yml)

## Design Notes

- Consumer repositories should keep thin local wrapper workflows.
- Local wrappers own the trigger (`workflow_dispatch`, `release`, etc.).
- Local wrappers map repository-specific secrets into the reusable workflows.
- Local wrappers keep stable workflow filenames for npm trusted publishing and
  repo-specific conventions.
- Dependency-update wrappers also own their schedules, permissions, package
  roots, and repository-specific validation. They consume the shared workflow
  by an immutable commit SHA and do not pass inherited secrets.
- Dependency and validation code runs in a read-only job. A fresh job receives
  only allowlisted manifests and lockfiles before it gets write permission.
- Shared workflows install exact `corepack@0.34.5`, select exact
  `pnpm@11.17.0`, and verify the active pnpm version before running package
  commands. The dependency updater also verifies that every package root
  declares the same pnpm version.

## Current Consumers

- `clevertask/kanban-board-ui`
- `clevertask/react-sortable-tree`
- `clevertask/scribe`
