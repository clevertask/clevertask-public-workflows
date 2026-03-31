# `create-release-version.yml`

Reusable workflow for bumping a package version, committing the change, tagging
the repository, pushing the commit/tag, and creating a GitHub release.

## Inputs

- `version`:
  Version to release, for example `0.1.0` or `1.0.0-alpha.0`.
- `prerelease`:
  When `true`, the GitHub release is marked as a prerelease.
- `working_directory`:
  Directory containing `package.json` and the optional `pnpm-lock.yaml`.
  Defaults to `.`.
- `node_version_file`:
  Path to the Node version file in the caller repository. Defaults to `.nvmrc`.

## Required Secrets

- `github_pat`:
  Personal access token used for checkout, push, and GitHub release creation.

## Example

```yaml
name: Create Release Version

on:
  workflow_dispatch:
    inputs:
      version:
        description: Version to release
        required: true
        type: string
      prerelease:
        description: Mark the GitHub release as a prerelease
        required: false
        default: false
        type: boolean

permissions:
  contents: write

jobs:
  release:
    uses: clevertask/clevertask-public-workflows/.github/workflows/create-release-version.yml@main
    with:
      version: ${{ inputs.version }}
      prerelease: ${{ inputs.prerelease }}
    secrets:
      github_pat: ${{ secrets.CLEVERTASK_PAT }}
```
