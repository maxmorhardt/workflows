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

Deployment is GitOps via **Argo CD**: workflows here don't apply anything to a cluster directly. The `cd-argocd` workflow bumps an image tag or chart version in the gitops repo (`maxmorhardt/k8s`) and Argo CD reconciles it. There is no `kubectl`/`helm upgrade` step, no cluster credentials, and no Tailscale in this repo.

## Setup

Secrets are passed per workflow (see each workflow's `secrets:` block), not inherited globally:

- `docker_username` / `docker_password` – Docker Hub login (Docker CI, when `push: true`).
- `release_please_token` – PAT or App token for release-please (Release Please).
- `gitops_token` – token with write access to the gitops repo (Argo CD).

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
      swagger: true

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
      image_tag: ${{ github.ref_name }}
    secrets:
      gitops_token: ${{ secrets.GITOPS_TOKEN }}
```

## Available Workflows

### CI

- **Go CI** (`ci-go.yml`) – `go vet`, golangci-lint, optional swagger check, unit + integration tests with the coverage gate (race enabled), Trivy filesystem scan, and optional cross-compiled (`amd64`/`arm64`) binary build. Go 1.26.
- **Node CI** (`ci-node.yml`) – pnpm install, type check, lint, `pnpm audit`, tests with coverage, build, and Trivy filesystem scan. Node 24.
- **Docker CI** (`ci-docker.yml`) – Buildx image build, Trivy image scan (fails on HIGH/CRITICAL), and optional multi-arch (`amd64`/`arm64`) push to Docker Hub with `latest`, provenance, and SBOM. Optionally consumes a prebuilt binary artifact.
- **Helm CI** (`ci-helm.yml`) – validates (via Helm Validate), packages the chart, and optionally pushes it as an OCI artifact to `ghcr.io/<owner>/charts`.

### Validation

- **Helm Validate** (`validate-helm.yml`) – renders a local or remote/OCI chart with `helm template`, optional `helm lint --strict`, and optional kubeconform validation of the rendered output.
- **Manifest Validate** (`validate-manifests.yml`) – validates raw Kubernetes manifests (files, dirs, or globs) with kubeconform against the default and CRD schema catalogs.

### Release & PR

- **Release Please** (`release-please.yml`) – runs release-please to maintain release PRs and tags from conventional commits.
- **PR Title** (`pr-title.yml`) – enforces a semantic (conventional-commit) pull request title.

### Deployment

- **Argo CD** (`cd-argocd.yml`) – GitOps deploy: bumps `image.tag` and/or the chart `targetRevision` in the Application manifest in the gitops repo, commits, and pushes (rebase-and-retry on conflict). Argo CD applies the change to the cluster.
