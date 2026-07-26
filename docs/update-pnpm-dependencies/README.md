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

Paths must exist inside the caller repository. Package roots must be unique,
must not overlap, and must contain regular, non-symlink `package.json`,
`pnpm-lock.yaml`, and `pnpm-workspace.yaml` files.

## Required caller setup

The caller repository must:

- grant `contents: write` and `pull-requests: write` to the calling job;
- allow GitHub Actions to create pull requests;
- declare pnpm in `packageManager` in every configured package root;
- omit `devEngines.packageManager` so it cannot conflict with the synchronized
  `packageManager`;
- define a non-empty `validate:deps` script in every configured package root;
- keep a Node LTS selector in the configured Node version file;
- configure update policy in each package root's `pnpm-workspace.yaml`.

The shared workflow synchronizes every `packageManager` field to its centrally
pinned pnpm version. Other caller workflows should install the exact pnpm
version from `packageManager` instead of repeating it elsewhere.

`validate:deps` must be non-interactive, exit nonzero on failure, and leave the
Git worktree clean. It should run the repository's formatting check, linter,
build, and real automated tests when they exist. Do not add dummy tests that
always pass. Callers should also run `validate:deps` from a `pull_request`
workflow when validation must appear as a check on the automation PR itself.

The agreed CleverTask package policy is:

```yaml
minimumReleaseAge: 4320

updateConfig:
  ignoreDependencies:
    - "@types/node"
```

The workflow intentionally reads the camelCase keys returned by the pinned
pnpm version for this policy: `minimumReleaseAge` and
`updateConfig.ignoreDependencies`.

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
4. verifies the Node runtime is LTS and every package root has pnpm,
   `validate:deps`, and the agreed release-age and Node-types policy;
5. synchronizes every `packageManager` field to `pnpm@11.17.0` and runs
   `pnpm up --latest` sequentially in every package root;
6. updates direct `@types/node` dependencies within the active Node LTS major;
7. rejects changes to scripts or any other non-dependency package fields
   besides the intentional `packageManager` synchronization;
8. exits successfully without an artifact or branch when dependencies did not
   change;
9. snapshots only package manifests and pnpm lockfiles into one immutable
   artifact;
10. performs a frozen install and runs `validate:deps` in every package root;
11. starts a fresh write-capable job, checks out the exact recorded SHA, and
   stops if the default branch moved or another automation PR appeared;
12. downloads the snapshot by artifact ID, independently verifies every path
    and non-dependency package field, and applies only the configured manifests
    and lockfiles;
13. uses the fresh write-capable job to push the unique branch and open the
    draft pull request only after the snapshot passes independent checks; and
14. uses a separate permissionless job to fail the workflow when validation
    failed.

Input, setup, or dependency-resolution failures happen before a reviewable
commit exists, so they fail the workflow without opening a pull request.
When `validate:deps` fails after a snapshot exists, the workflow still opens
the draft pull request with a failure note, then fails the run so the proposed
dependency changes remain available for diagnosis.
When the default branch moves during preparation, rerun the full workflow so a
new snapshot is built from the current base.

The workflow uses only the caller's `GITHUB_TOKEN`; callers must not use
`secrets: inherit`. The schedule or manual validation run belongs to the
default-branch event rather than the new PR head, so the draft PR body links to
that run. Separate pull-request checks triggered by the token require manual
approval from a repository writer. A narrowly scoped GitHub App can replace
this later if manual approval becomes too costly.

The read-only and write-capable jobs are intentionally isolated. GitHub grants
permissions at job scope, so the publish job is write-capable from its start;
package lifecycle and validation code never run there. The artifact is treated
as untrusted input: the publish job downloads it by immutable ID, checks its
digest through `actions/download-artifact`, rejects non-regular or unexpected
paths, and never executes code from it. Checkout credentials are not persisted,
and Git or GitHub CLI write operations happen only after these checks pass.

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
```

Do not add automatic merge, publish, or deploy steps to this wrapper. The draft
pull request is the review and recovery boundary.

## Toolchain maintenance

The shared dependency workflow is the source of truth for the exact pnpm
version. Change that pin in an explicit reviewed pull request. After a consumer
updates its immutable reusable-workflow SHA, the next dependency run
synchronizes `packageManager` and the caller's package workflows select that
version through Corepack.

Use Dependabot's `github-actions` ecosystem in this repository and every
consumer. It updates third-party Action references and called reusable-workflow
SHAs without taking ownership of npm manifests or pnpm lockfiles:

```yaml
version: 2

updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      github-actions:
        patterns:
          - "*"
```

Do not add a Dependabot npm version-update entry as part of this process.
Change the pnpm or Corepack pin through an explicit manual pull request. Update
GitHub Action and reusable-workflow references through reviewed Dependabot pull
requests.
