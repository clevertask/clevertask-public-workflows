# `create-release-version.yml`

Reusable workflow for bumping a package version, committing the change, tagging
the repository, pushing the commit/tag, and creating a GitHub release.

## Inputs

- `version`:
  Version to release, for example `0.1.0`, `1.0.0-alpha.1`, or
  `1.0.0-beta.1`.
- `prerelease`:
  When `true`, the GitHub release is marked as a prerelease.
- `working_directory`:
  Directory containing `package.json` and the optional `pnpm-lock.yaml`.
  Defaults to `.`.
- `node_version_file`:
  Path to the Node version file in the caller repository. Defaults to `.nvmrc`.

## Required Secrets

- `github_pat`:
  Personal access token used to push the version commit and tag, then create
  the GitHub release.

## Notes

- This workflow installs exact `corepack@0.34.5`, selects and verifies the
  exact pnpm version declared by the package's `packageManager`, and then
  changes package metadata.
- `packageManager` is the only package-manager declaration;
  `devEngines.packageManager` must not be set.
- It validates the working directory and Node version file before toolchain
  setup and does not install dependencies.
- Versioning disables `preversion`, `version`, and `postversion` scripts. Git
  hooks are disabled for the commit and push.
- Versions may use SemVer prerelease identifiers such as `-alpha.1` or
  `-beta.1`. A leading `v` and SemVer build metadata such as `+build.7` are
  rejected because the pinned pnpm release and packing path does not preserve
  build metadata.
- Only `package.json` and an optional `pnpm-lock.yaml` may change. The workflow
  verifies the requested version, unchanged package-manager declaration,
  staged paths, committed paths, and a clean worktree.
- Checkout credentials are not persisted. The PAT is exposed only to the push
  and GitHub-release steps.
- Releases must run from a branch. The version commit and tag are pushed
  atomically before the GitHub release is created.
- A rerun checks out the current branch head. If the atomic push succeeded but
  GitHub release creation failed, the workflow verifies that the existing tag,
  branch head, and package version agree, then resumes at release creation. If
  the GitHub release already exists, the rerun exits successfully.

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
  contents: read

jobs:
  release:
    uses: clevertask/clevertask-public-workflows/.github/workflows/create-release-version.yml@<FULL_COMMIT_SHA>
    with:
      version: ${{ inputs.version }}
      prerelease: ${{ inputs.prerelease }}
    secrets:
      github_pat: ${{ secrets.CLEVERTASK_PAT }}
```
