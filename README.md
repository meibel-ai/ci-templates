# ci-templates

> **⚠️ THIS REPO IS PUBLIC.** Contents are readable by anyone on the internet. Do NOT commit:
> - Secrets, tokens, API keys, private URLs, or any credential material.
> - Proprietary business logic, customer data references, or internal-only knowledge.
> - Anything that would embarrass Meibel if screenshotted on Twitter.
>
> Everything here is public because these workflows are consumed by every domain repo via `uses: meibel-ai/ci-templates/...@v1` and cross-repo private-repo access requires PAT/App-token gymnastics. Public keeps `actions/checkout` and reusable-workflow calls simple. See [platform-workflow LOG 2026-07-18 evening 10](../marvin-repo/marvin/blob/main/docs/platform-workflow/LOG.md) for the decision context.
>
> **What IS safe here:** reusable workflow orchestration (build/lint/test/sign/promote), JSON schemas for `service.yaml`, secret-input NAMES (never values), GCP WIF resource paths (public identifiers), runner labels, image path patterns. All auth is bound on Meibel's side (WIF IAM bindings, GitHub App installations) — the paths alone grant nothing.

Shared reusable GitHub Actions workflows for all meibel-ai domain repos.

This repo is Tier-1 — changes must use immutable SemVer tags only.

## Usage

```yaml
# .github/workflows/ci.yml (in your service repo)
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    uses: meibel-ai/ci-templates/.github/workflows/build-and-promote.yml@v1.0.0
    with:
      service-name: my-service
      language: python          # python | node | go | rust
      runs-on: meibel-eks-dev   # meibel-eks-dev (CPU) | super-runner (GPU)
    secrets: inherit
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `service-name` | yes | — | Used for image name and gitops overlay path |
| `language` | yes | — | `python`, `node`, `go`, or `rust`. Fails immediately if absent or invalid. |
| `runs-on` | yes | — | `meibel-eks-dev` for CPU, `super-runner` for GPU |
| `registry` | no | `ghcr.io/meibel-ai` | Container registry prefix |
| `wif-provider` | no | core pool | Full WIF provider path. Override for hub-pool GPU services. |
| `cache-type` | no | `gha` | Use `registry` for GPU repos where images exceed 10GB GHA cache limit |
| `docker-context` | no | `service-path` | Docker build context. Set `.` for apps in a JS workspace whose image needs the repository root (shared libs, the root lockfile). |
| `dockerfile` | no | `<service-path>/Dockerfile` | Dockerfile path, relative to the repository root. |
| `node-package-manager` | no | `npm` | `npm` (unchanged) or `pnpm` for apps in a pnpm workspace: corepack pnpm, `pnpm install --frozen-lockfile` at `node-workspace-root`, build the service's workspace dependencies, then the service's `lint` and `test` scripts. |
| `node-workspace-root` | no | `.` | pnpm only: directory with `pnpm-workspace.yaml` and `pnpm-lock.yaml`. |

## WIF pools

- **Core pool** (default): `projects/502352286495/.../meibel-ai-gh-wd-pool/providers/meibel-ai-gh-wd-pool-provider`
- **Hub pool**: `projects/1043014246178/.../meibel-gh-wd-pool--um0j/providers/meibel-ai-gh-wd-pool-provider`

## Cosign signing identity

The signing subject is this workflow file URL, not the caller's:

```
https://github.com/meibel-ai/ci-templates/.github/workflows/build-and-promote.yml
```

Kyverno `verifyImages` policy must reference this path.

## Promote step prerequisite

The gitops-platform promote step requires WS-D1 (directory restructure) to be complete.
The overlay path it writes to: `apps/workloads/{service-name}/envs/dev/kustomization.yaml`

## pnpm workspace apps

For an app in a pnpm workspace (for example `platform-ui/services/nexus-web`), build the image from the repository root:

```yaml
uses: meibel-ai/ci-templates/.github/workflows/build-and-promote.yml@v1
with:
  service-name: nexus-web
  language: node
  service-path: services/nexus-web
  node-package-manager: pnpm
  docker-context: .
  dockerfile: services/nexus-web/Dockerfile
```

The app's Dockerfile prunes the workspace to the app (`turbo prune <package> --docker`) and installs from the pruned lockfile. The root `package.json` sets `packageManager` (for example `pnpm@9.15.0`), which corepack uses.

