# Shared release CI

Reusable release workflows + a SemVer composite action, so every Onsetto repo
runs the **same** release logic instead of drifting copies.

## Pieces

| File | Purpose |
|---|---|
| `.github/actions/resolve-release-version/action.yml` | Canonical SemVer resolution + release-model rules (main / release/* / hotfix/*), `tag_type` validation, and optional onsetto-api ref resolution. Single source of truth. |
| `.github/workflows/release-container.yml` | Reusable: build a Docker image → push to ECR `<repo>:<version>` → GitHub release. EKS API + Lambda images. |
| `.github/workflows/release-static.yml` | Reusable: resolve version → GitHub release (tag-only; static apps build at deploy time). |

Each run prints the resolved version in its **job summary** (`✅ Released vX —
Deploy with: vX`), so you can grab the exact tag for the deploy at a glance. Both
reusable workflows also expose `release_number` and `release_class` as
**workflow outputs** if you ever want to chain release → deploy.

## Caller examples (in each app repo's `.github/workflows/release-*.yml`)

Callers must declare `permissions:` — a reusable workflow is capped by the
caller's token, and the org default is read-only. The `run-name` gives each run a
consistent, scannable title.

**Container, no upstream copy (onsetto-api, onsetto-document-processor):**
```yaml
name: Release API Image (SemVer)
run-name: "Release · ${{ github.ref_name }} · ${{ inputs.version != '' && inputs.version || (github.ref_name == 'main' && format('stable (bump:{0})', inputs.version_bump) || inputs.tag_type) }} · @${{ github.actor }}"
on:
  workflow_dispatch:
    inputs:
      version: { description: "Exact version (blank for auto)", required: false }
      version_bump: { type: choice, required: true, default: "-", options: ["-", patch, minor, major] }
      tag_type: { type: choice, required: true, default: stable, options: [candidate, stable] }
permissions:
  contents: write   # create the tag + GitHub release
  id-token: write   # OIDC for AWS
jobs:
  release:
    uses: Join-Roy/.github/.github/workflows/release-container.yml@main
    secrets: inherit
    with:
      version: ${{ inputs.version }}
      version_bump: ${{ inputs.version_bump }}
      tag_type: ${{ inputs.tag_type }}
```

**Container, copies onsetto-api (transaction-processor, email-notifier)** — also
add an `onsetto_api_ref` input and set `copy_onsetto_api: true`:
```yaml
    with:
      version: ${{ inputs.version }}
      version_bump: ${{ inputs.version_bump }}
      tag_type: ${{ inputs.tag_type }}
      onsetto_api_ref: ${{ inputs.onsetto_api_ref }}
      copy_onsetto_api: true
```

**Static (onsetto-ui, onsetto-switch-hub):**
```yaml
run-name: "Release · ${{ github.ref_name }} · ${{ inputs.version != '' && inputs.version || (github.ref_name == 'main' && format('stable (bump:{0})', inputs.version_bump) || inputs.tag_type) }} · @${{ github.actor }}"
permissions:
  contents: write   # create the tag + GitHub release
jobs:
  release:
    uses: Join-Roy/.github/.github/workflows/release-static.yml@main
    secrets: inherit
    with:
      version: ${{ inputs.version }}
      version_bump: ${{ inputs.version_bump }}
      tag_type: ${{ inputs.tag_type }}
```

## Repo → workflow mapping

| Repo | Reusable workflow | `copy_onsetto_api` |
|---|---|---|
| onsetto-api | release-container | false |
| onsetto-document-processor | release-container | false |
| onsetto-transaction-processor | release-container | true |
| onsetto-email-notifier | release-container | true |
| onsetto-ui | release-static | — |
| onsetto-switch-hub | release-static | — |

## What this reconciles (vs. the old per-repo copies)

- **One version-resolution implementation** (the newest logic, incl. re-runnable
  hotfix releases) — replaces three drifted generations across the repos.
- **Slack release notifications removed** (was only in document-processor).
- **onsetto-api fetch standardized** to checkout + `GH_ACCESS_TOKEN` + ref
  resolution (document-processor previously used an SSH deploy key — now removed
  entirely, since it doesn't actually copy onsetto-api).
- **ECR repo derived** from the repo name (was hardcoded in document-processor).
- **Buildx** standardized to inline `docker buildx create`.

## Dependencies / setup

- **Caller permissions (required):** the caller must declare `permissions:`
  (`contents: write`, plus `id-token: write` for container builds). A reusable
  workflow's permissions are capped by the caller, and the org default is read-only.
- **Org Actions setting:** `Join-Roy/.github` is public, so its reusable workflows
  are callable by private repos with no extra access policy.
- **Secrets:** each caller needs `GH_ACCESS_TOKEN` (passed via `secrets: inherit`).
- **Repo/org variables:** `ROOT_ACCOUNT_AWS_REGION`, `ROOT_ACCOUNT_AWS_ACCOUNT_ID`,
  `ROOT_ACCOUNT_AWS_ROLE_ARN` (already used by the existing workflows).
- **Pinning:** examples use `@main`; pin callers (and the in-workflow action
  reference) to a release tag of this repo (e.g. `@v1`) once validated.
