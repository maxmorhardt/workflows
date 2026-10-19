# Workflows

![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Argo CD](https://img.shields.io/badge/argo%20cd-%23EF7B4D.svg?style=for-the-badge&logo=argo&logoColor=white)
![Helm](https://img.shields.io/badge/helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![Go](https://img.shields.io/badge/go-%2300ADD8.svg?style=for-the-badge&logo=go&logoColor=white)
![Node.js](https://img.shields.io/badge/node.js-6DA55F?style=for-the-badge&logo=node.js&logoColor=white)

Reusable GitHub Actions workflows for CI/CD.

## Overview

This repository contains reusable GitHub Actions workflows that can be called from other repositories to standardize CI/CD. They cover building and scanning Docker images, testing Go and Node apps, packaging Helm charts, validating manifests, cutting releases, and deploying.

Deployment is GitOps via **Argo CD**: workflows here don't apply anything to a cluster directly. The `cd-argocd` workflow bumps an image tag or chart version for one cluster (`dev` or `prod`) in the gitops repo (`maxmorhardt/gitops`) and Argo CD reconciles it; prod waits for approval on the `production` environment. There is no `kubectl`/`helm upgrade` step, no cluster credentials, and no Tailscale in this repo.

## Setup

Secrets are passed per workflow (see each workflow's `secrets:` block), not inherited globally:

- `docker_username` / `docker_password` – Docker Hub login (Docker CI, when `push: true`).
- `release_please_token` – PAT or App token for release-please (Release Please).
- `gitops_token` – token with write access to the gitops repo (Argo CD).
- `aws_access_key_id` / `aws_secret_access_key` – AWS credentials for the Terraform workflows. Required by every module, since all state lives in one S3 backend.
- `cloudflare_api_token` / `repo_admin_token` – provider credentials for the Terraform Cloudflare and GitHub modules.
- `tf_vars` – tfvars content written to `ci.auto.tfvars` before plan or apply, for values kept out of git.

The Helm CI and PR Title workflows use the built-in `GITHUB_TOKEN`.

## Usage

Reference a workflow with `uses:` and pass its inputs/secrets. Example — Go CI plus a push-and-deploy on `main`:

```yaml
name: CI

on:
  push:
    branches: [main]

jobs:
  ci:
    uses: maxmorhardt/workflows/.github/workflows/ci-go.yml@main
    with:
      repository: ${{ github.repository }}

  docker:
    needs: ci
    uses: maxmorhardt/workflows/.github/workflows/ci-docker.yml@main
    with:
      repository: ${{ github.repository }}
      app_name: my-app
      app_version: ${{ github.ref_name }}
      push: true
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_password: ${{ secrets.DOCKER_PASSWORD }}

  deploy:
    needs: docker
    uses: maxmorhardt/workflows/.github/workflows/cd-argocd.yml@main
    with:
      app: my-app
      environment: prod
      image_tag: 1.4.0
      chart_version: 1.4.0
    secrets:
      gitops_token: ${{ secrets.GITOPS_TOKEN }}
```

## Available Workflows

### CI

- **Go CI** (`ci-go.yml`) – `go vet`, golangci-lint, the swagger check when the module's Makefile has `swag-check`, unit + integration tests with the coverage gate (race enabled), and Trivy filesystem scan. Go version from the module's `go.mod`. `working_directory` points it at a module inside a monorepo. Images compile Go in their own multi-stage Dockerfile, so it builds no binaries.
- **Node CI** (`ci-node.yml`) – pnpm install, type check, lint, `pnpm audit`, tests with coverage, build, and Trivy filesystem scan. Node 24. `working_directory` points it at an app inside a monorepo.
- **Docker CI** (`ci-docker.yml`) – Lints the Dockerfile with hadolint (fails on warnings; `DL3018` apk pinning is ignored), builds once to an OCI archive with provenance and SBOM (`amd64` and `arm64`), Trivy-scans each platform of that archive (fails on HIGH/CRITICAL), then pushes the scanned bytes to Docker Hub as the given version and `latest`. Optionally consumes a build artifact, such as a Node app's `dist/`. `context`, `dockerfile`, and `build_args` support monorepos and one Dockerfile building several images.
- **Helm CI** (`ci-helm.yml`) – validates (via Helm Validate), then packages the chart and pushes it to `oci://ghcr.io/<owner>/charts`. Name and version come from `Chart.yaml`; `version` overrides the version (e.g. `0.0.0-sha-<commit>` for dev).

### Validation

- **Helm Validate** (`validate-helm.yml`) – renders a local or remote/OCI chart with `helm template` (one or more `values_files`), optional `helm lint --strict`, and always validates the output with strict kubeconform against the default and CRD schema catalogs (CRD definitions themselves are skipped).
- **Manifest Validate** (`validate-manifests.yml`) – validates raw Kubernetes manifests (files, dirs, or globs) with strict kubeconform against the default and CRD schema catalogs; every kind must have a schema.
- **Terraform Validate** (`validate-terraform.yml`) – `fmt -check`, init, validate, and plan for one root module (`working_directory`), with the plan posted to the job summary. `fail_on_diff: true` turns a non-empty plan into a failure, which is how scheduled drift detection works.

### Release & PR

- **Release Please** (`release-please.yml`) – runs release-please to maintain release PRs and tags from conventional commits.
- **PR Title** (`pr-title.yml`) – enforces a semantic (conventional-commit) pull request title.

### Deployment

- **Argo CD** (`cd-argocd.yml`) – GitOps deploy: bumps `image.tag` and/or the chart `targetRevision` in `clusters/<environment>/apps/<app>.yaml` in the gitops repo, commits, and pushes (rebase-and-retry on conflict). `environment: prod` runs in the `production` environment, so it waits for approval; `dev` runs in `development`. Argo CD applies the change to the cluster.
- **Promote image** (`cd-promote.yml`) – waits for an image tag already built and tested (e.g. dev's `sha-<commit>`), then retags that exact multi-arch manifest as the release version and `stable` without rebuilding, so prod runs the bytes dev ran and `stable` is always what prod runs.
- **Terraform Apply** (`cd-terraform.yml`) – plans one root module (`working_directory`) to a file and applies that exact file, so the apply cannot recompute into a different change than the one planned. The diff is reviewed on the pull request, not at apply time. Runs in the `production` environment.
