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
- It installs exact `corepack@0.34.5`, selects and verifies the exact pnpm
  version declared by the package's `packageManager`, requires pnpm 11.1.3 or
  newer for trusted publishing, and installs dependencies with
  `--frozen-lockfile`.
- `packageManager` is the only package-manager declaration;
  `devEngines.packageManager` must not be set.
- A read-only preparation job runs formatting, the build, `prepublishOnly` when
  present, and `pnpm pack`. Package lifecycle code has no npm OIDC token.
- Only the resulting tarball crosses into a fresh OIDC-enabled job. That job
  has no checkout, verifies the artifact identity, filename, and tarball
  SHA-256, then publishes with `--ignore-scripts` and `--no-git-checks`.
- `prepack`, `prepare`, and `postpack` may run during packing in the read-only
  job. `publish` and `postpublish` scripts do not run in the privileged job.
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
    uses: clevertask/clevertask-public-workflows/.github/workflows/publish-npm.yml@<FULL_COMMIT_SHA>
    with:
      dist_tag: ${{ inputs.dist_tag }}
```
