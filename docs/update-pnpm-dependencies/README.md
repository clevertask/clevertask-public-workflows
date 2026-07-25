# `update-pnpm-dependencies.yml`

Reusable workflow for updating pnpm dependencies and opening one draft pull
request in the caller repository.

## Inputs

- `package_roots`:
  Required JSON array of package roots relative to the caller repository. Use
  `["."]` for a single-package repository or values such as
  `["client","server","mcp"]` for independent package roots.
- `node_version_file`:
  Node version file relative to the caller repository. Defaults to `.nvmrc`.
- `validation_script`:
  Caller-owned validation script relative to the repository root. Defaults to
  `.github/scripts/validate-dependency-update.sh`.

Paths must exist inside the caller repository. Package roots must be unique and
contain both `package.json` and `pnpm-lock.yaml`.

## Required caller setup

The caller repository must:

- grant `contents: write` and `pull-requests: write` to the calling job;
- allow GitHub Actions to create pull requests;
- declare `packageManager: "pnpm@11.17.0"` in every configured package root;
- keep a Node LTS selector in the configured Node version file;
- provide a validation script that exits nonzero on failure and leaves the Git
  worktree clean; and
- configure update policy in each package root's `pnpm-workspace.yaml`.

Callers should also run the same validation script from a `pull_request`
workflow when validation must appear as a check on the automation PR itself.

The agreed CleverTask package policy is:

```yaml
minimumReleaseAge: 4320

updateConfig:
  ignoreDependencies:
    - "@types/node"
```

An explicit `minimumReleaseAge` is strict by default in pnpm 11. This means an
update can fail when no eligible version is at least three days old. Keep
repository-specific release-age exclusions, overrides, and audit decisions in
the caller repository.

## Behavior

The workflow:

1. skips successfully if an open pull request already uses the
   `chore/weekly-dependency-update-` branch prefix;
2. records the caller's default-branch SHA and performs the update in a
   read-only job with no persisted Git credentials;
3. installs exact `corepack@0.34.5` and `pnpm@11.17.0`;
4. verifies the Node runtime is LTS and every package root has the agreed pnpm,
   release-age, and Node-types policy;
5. runs `pnpm up --latest` sequentially in every package root;
6. updates direct `@types/node` dependencies within the active Node LTS major;
7. exits successfully without an artifact or branch when dependencies did not
   change;
8. snapshots only package manifests and pnpm lockfiles into one immutable
   artifact before running the caller-owned validation script;
9. starts a fresh write-capable job, checks out the exact recorded SHA, and
   stops if the default branch moved or another automation PR appeared;
10. downloads the snapshot by artifact ID, independently verifies every path,
    and applies only the configured manifests and lockfiles;
11. introduces the write token only while pushing the unique branch and opening
    the draft pull request; and
12. uses a separate permissionless job to fail the workflow when validation
    failed.

Input, setup, or dependency-resolution failures happen before a reviewable
commit exists, so they fail the workflow without opening a pull request.
When the default branch moves during preparation, rerun the full workflow so a
new snapshot is built from the current base.

The workflow uses only the caller's `GITHUB_TOKEN`; callers must not use
`secrets: inherit`. The schedule or manual validation run belongs to the
default-branch event rather than the new PR head, so the draft PR body links to
that run. Separate pull-request checks triggered by the token require manual
approval from a repository writer. A narrowly scoped GitHub App can replace
this later if manual approval becomes too costly.

The read-only and write-capable jobs are intentionally isolated. Package
lifecycle and validation code never run in the job that receives
`contents: write`. The artifact is treated as untrusted input: the publish job
downloads it by immutable ID, checks its digest through
`actions/download-artifact`, rejects non-regular or unexpected paths, and never
executes code from it.

## Outputs

- `changed`: `true` when dependency files changed.
- `pull_request_url`: the existing or newly created automation pull request.
- `skipped`: `true` when an earlier automation pull request is still open.
- `validation_status`: `success`, `failure`, or `skipped`.

## Caller example

Keep the schedule, manual recovery trigger, permissions, concurrency, package
roots, and validation logic in the caller repository. Pin the reusable workflow
to the full commit SHA selected for rollout.

```yaml
name: Weekly Dependency Update

on:
  schedule:
    - cron: "17 10 * * 2"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: weekly-dependency-update
  cancel-in-progress: false

jobs:
  update:
    uses: clevertask/clevertask-public-workflows/.github/workflows/update-pnpm-dependencies.yml@<FULL_COMMIT_SHA>
    with:
      package_roots: '["."]'
      validation_script: ".github/scripts/validate-dependency-update.sh"
```

Do not add automatic merge, publish, or deploy steps to this wrapper. The draft
pull request is the review and recovery boundary.

## Toolchain maintenance

This workflow intentionally does not update its own pnpm, Corepack, Action, or
reusable-workflow references. Maintain those pins in explicit pull requests.
Dependabot can own the `github-actions` ecosystem without owning npm manifests
or pnpm lockfiles.
