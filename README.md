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

## Current Consumers

- `clevertask/scribe`

More repositories can be migrated as the shared contracts settle.
