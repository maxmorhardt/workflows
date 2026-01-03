# Workflows

![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![Go](https://img.shields.io/badge/go-%2300ADD8.svg?style=for-the-badge&logo=go&logoColor=white)
![Node.js](https://img.shields.io/badge/node.js-6DA55F?style=for-the-badge&logo=node.js&logoColor=white)

Reusable GitHub Actions workflows for CI/CD.

## Overview

This repository contains reusable GitHub Actions workflows that can be called from other repositories to standardize and simplify CI/CD processes. These workflows handle common tasks like building Docker images, running tests, and deploying to Kubernetes clusters.

## Setup

Required secrets in calling repositories:
- `DOCKER_USERNAME` - Docker Hub username
- `DOCKER_PASSWORD` - Docker Hub password/token
- `KUBE_CONFIG` - Base64-encoded kubeconfig file

## Usage

To use a workflow from this repository, reference it in your workflow file:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: your-org/workflows/.github/workflows/ci-docker.yml@main
    secrets: inherit
```

## Available Workflows

### CI Workflows
- **Node.js CI** - Runs tests and linting for Node.js projects
- **Go CI** - Runs tests and builds for Go applications
- **Docker CI** - Builds and tests Docker images
- **Helm CI** - Lints and validates Helm charts

### Deployment Workflows
- **Deploy with Helm** - Deploys applications to Kubernetes using Helm charts