# Workflows

Reusable GitHub Actions workflows for CI/CD.

## Setup

Required secrets in calling repositories:
- `DOCKER_USERNAME` - Docker Hub username
- `DOCKER_PASSWORD` - Docker Hub password/token
- `KUBE_CONFIG` - Base64-encoded kubeconfig file