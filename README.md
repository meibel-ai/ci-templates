# ci-templates

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
