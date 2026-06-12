# Shared release CI

Reusable release workflows + a SemVer composite action, so every Onsetto repo
runs the **same** release logic instead of drifting copies.

## Pieces

| File | Purpose |
|---|---|
| `.github/actions/resolve-release-version/action.yml` | Canonical SemVer resolution + release-model rules (main / release/* / hotfix/*), `tag_type` validation, and optional onsetto-api ref resolution. Single source of truth. |
| `.github/workflows/release-container.yml` | Reusable: build a Docker image → push to ECR `<repo>:<version>` → GitHub release. EKS API + Lambda images. |
| `.github/workflows/release-static.yml` | Reusable: resolve version → GitHub release (tag-only; static apps build at deploy time). |

## Caller examples (in each app repo's `.github/workflows/release-*.yml`)

**Container, no upstream copy (onsetto-api):**
```yaml
name: Release API Image (SemVer)
on:
  workflow_dispatch:
    inputs:
      version: { description: "Exact version (blank for auto)", required: false }
      version_bump: { type: choice, required: true, default: "-", options: ["-", patch, minor, major] }
      tag_type: { type: choice, required: true, default: stable, options: [candidate, stable] }
jobs:
  release:
    uses: Join-Roy/.github/.github/workflows/release-container.yml@main
    secrets: inherit
    with:
      version: ${{ inputs.version }}
      version_bump: ${{ inputs.version_bump }}
      tag_type: ${{ inputs.tag_type }}
```

**Container, copies onsetto-api (the three Lambda repos)** — add the `onsetto_api_ref`
input and set `copy_onsetto_api: true`:
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
| onsetto-document-processor | release-container | true |
| onsetto-transaction-processor | release-container | true |
| onsetto-email-notifier | release-container | true |
| onsetto-ui | release-static | — |
| onsetto-switch-hub | release-static | — |

## What this reconciles (vs. the old per-repo copies)

- **One version-resolution implementation** (the newest logic, incl. re-runnable
  hotfix releases) — replaces three drifted generations across the repos.
- **Slack release notifications removed** (was only in document-processor).
- **onsetto-api fetch standardized** to checkout + `GH_ACCESS_TOKEN` + ref
  resolution (document-processor previously used an SSH deploy key).
- **ECR repo derived** from the repo name (was hardcoded in document-processor).
- **Buildx** standardized to inline `docker buildx create`.

## Dependencies / setup

- **Org Actions setting:** allow private repos to use workflows from `Join-Roy/.github`
  (Org → Settings → Actions → "Accessible from repositories owned by the organization").
- **Secrets:** each caller needs `GH_ACCESS_TOKEN` (passed via `secrets: inherit`).
  document-processor must gain `GH_ACCESS_TOKEN` and can drop `DOCUMENT_PARSER_DEPLOY_KEY`.
- **Repo/org variables:** `ROOT_ACCOUNT_AWS_REGION`, `ROOT_ACCOUNT_AWS_ACCOUNT_ID`,
  `ROOT_ACCOUNT_AWS_ROLE_ARN` (already used by the existing workflows).
- **Pinning:** examples use `@main`; pin callers and the in-workflow action checkout
  to a release tag of this repo (e.g. `@v1`) once validated.
