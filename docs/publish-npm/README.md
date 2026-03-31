# `publish-npm.yml`

Reusable workflow for publishing a pnpm-based package to npm.

## Inputs

- `dist_tag`:
  npm dist-tag to publish under, for example `latest` or `next`. Defaults to
  `latest`.
- `working_directory`:
  Directory containing the package to publish. Defaults to `.`.
- `node_version_file`:
  Path to the Node version file in the caller repository. Defaults to `.nvmrc`.

## Notes

- This workflow is intended to be called from a local wrapper workflow in the
  consumer repository.
- It uses `pnpm publish --no-git-checks` so detached-HEAD CI contexts do not
  block releases.
- For npm trusted publishing, configure the trusted publisher against the local
  wrapper workflow filename in the consumer repository.

## Example

```yaml
name: Publish Package

on:
  workflow_dispatch:
    inputs:
      dist_tag:
        description: npm dist-tag to publish under
        required: true
        default: latest
        type: choice
        options:
          - latest
          - next

permissions:
  id-token: write
  contents: read

jobs:
  publish:
    uses: clevertask/clevertask-public-workflows/.github/workflows/publish-npm.yml@main
    with:
      dist_tag: ${{ inputs.dist_tag }}
```
