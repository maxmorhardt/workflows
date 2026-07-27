# AGENTS

## Project Context

- Project: Workflows (`github.com/maxmorhardt/workflows`)
- Language: YAML (GitHub Actions reusable workflows), plus JSON for the shared Renovate preset
- Purpose: The shared CI/CD library for every `maxmorhardt` repo. Reusable workflows called with `uses:` from other repositories.
- Target environments: GitHub-hosted `ubuntu-latest` runners. Deployment is GitOps, so the deploy workflow commits to the gitops repo and Argo CD reconciles.
- Related repos: every repo in the org consumes these. `k8s` is the gitops target of `cd-argocd.yml`.

**Consumers pin `@main`, not a tag.** A merge here is live for every repo on its next run.

## Repository Layout

- `.github/workflows/` - the reusable workflows. Every one is `on: workflow_call`.
  - `ci-go.yml` - `go vet`, golangci-lint, optional swagger check, unit and integration tests with the coverage gate (race on), Trivy filesystem scan, optional cross-compiled `amd64`/`arm64` binary. Go 1.26.
  - `ci-node.yml` - pnpm install, type check, lint, `pnpm audit`, tests with coverage, build, Trivy filesystem scan. Node 24.
  - `ci-docker.yml` - Buildx build, Trivy image scan failing on HIGH and CRITICAL, optional multi-arch push to Docker Hub with `latest`, provenance, and SBOM.
  - `ci-helm.yml` - validates, packages, and optionally pushes a chart as an OCI artifact to `ghcr.io/<owner>/charts`.
  - `validate-helm.yml` - renders a local or remote OCI chart, optional `helm lint --strict`, optional kubeconform on the output.
  - `validate-manifests.yml` - kubeconform over raw manifests (files, dirs, globs) against the default and CRD schema catalogs.
  - `release-please.yml` - maintains release PRs and tags from conventional commits.
  - `pr-title.yml` - enforces a semantic PR title.
  - `cd-argocd.yml` - GitOps deploy. Bumps `image.tag` and/or chart `targetRevision` in the `Application` manifest in `maxmorhardt/k8s`, commits, pushes with rebase-and-retry.
- `renovate/default.json` - the shared Renovate preset every repo extends via a two-line `renovate.json`.
- `renovate/README.md` - what the preset does and the one-time GitHub App setup.

## Core Principles

1. Every change ships to every consumer immediately
   - Treat each edit as a breaking-change review. Adding a required input, renaming an input or secret, or changing a default breaks callers silently.
2. Backward compatibility is a contract
   - New behavior arrives as an **optional** input with a default that preserves current behavior. Opt repos in one at a time.
3. CI never holds cluster credentials
   - No workflow here may hold a kubeconfig, cluster credentials, or Tailscale auth. If a change seems to need one, the answer is a commit to the gitops repo instead.
4. Secrets are explicit, never ambient
   - Secrets are declared and passed per workflow in each `secrets:` block, not inherited globally by default.
5. Security gates do not bend
   - Trivy scans fail on HIGH and CRITICAL. Lowering the threshold to get a build green is not a fix.
6. Policy lives in one place
   - Language and tool versions, and Renovate policy, are set here once for every repo. Per-repo exceptions belong in that repo, not in the shared preset.

## Agent Instructions

- Make the smallest safe change that solves the requested problem.
- One workflow per file, named `<kind>-<target>.yml`: `ci-` for build and test, `validate-` for check-only, `cd-` for deploy, plus the two release and PR helpers.
- Inputs are `snake_case` (`app_name`, `chart_directory`, `image_tag`, `working_directory`). Follow existing names rather than introducing synonyms.
- Pin third-party actions to a major version tag (`actions/checkout@v7`, `aquasecurity/trivy-action@v0.36.0`) so Renovate can bump them.
- Current secrets: `docker_username` and `docker_password` (Docker CI push), `release_please_token` (release-please), `gitops_token` (Argo CD deploy). Helm CI and PR Title use the built-in `GITHUB_TOKEN`.
- Bumping Go 1.26 or Node 24 is a fleet-wide change. Check that consumers' `go.mod` and `package.json` engines agree first.
- Renovate policy changes go in `renovate/default.json`. Per-repo exceptions belong in that repo's `renovate.json` as a `packageRules` entry. Never special-case a single repo in the shared preset.
- Keep `README.md` in sync. It documents each workflow's inputs and is what people read before wiring one up.

## Change Safety Checklist

Before merging a change to any workflow:

- Is every new input optional, with a default preserving current behavior?
- Did any input, secret, or output get renamed or removed? If so, which repos break?
- Grep the sibling workspaces for callers and confirm their inputs still line up:
  ```bash
  grep -rn "workflows/.github/workflows/ci-go.yml" ../*/.github/workflows/
  ```
- Does the change add a secret requirement? Every calling repo must have that secret set before merge.
- Does it introduce a credential this repo should not hold?
- Is `README.md` updated to match?

## Testing Guidance

There is no local test harness. A workflow is only truly exercised when a consumer runs it, which is exactly why the safety checklist above matters.

- Validate syntax with `actionlint` before pushing.
- Test a risky change on a branch first: point one consumer repo at `@<branch>` temporarily, confirm a green run, then merge here and revert the consumer to `@main`.
- Prefer a change that is inert unless a new input is set. That way a mistake affects nothing until someone opts in.
- After merging, watch the next run in at least one consuming repo per affected language rather than assuming success.
- For `cd-argocd.yml` changes, confirm the commit it produces in `k8s` is correct before trusting it. A bad bump reconciles straight into the cluster.

## Renovate Preset Behavior

Worth knowing because it shapes commit traffic in every repo:

- Minor and patch updates batch into one rolling PR per repo as `chore(deps):`, which cuts no release.
- Majors get individual PRs.
- Vulnerability alerts open immediately as `fix(deps):`, so release-please cuts a patch release and the fix deploys.
- `minimumReleaseAge: 3 days` guards against fresh-release supply-chain attacks.
- `gomodTidy` runs after every `go.mod` update.
- A Dependency Dashboard issue in each repo lists every pending and ignored update.

## Commit Tags

Conventional commits, enforced on PR titles by `pr-title.yml`.

- `feat`: New workflow, or a new capability on an existing one.
- `fix`: Corrects a broken or unsafe workflow behavior.
- `chore`: Maintenance that does not change workflow behavior, including routine action bumps.
- `ci`: Changes to this repo's own automation.
- `docs`: `README.md` only.

Scope is the workflow or area.

Example commit subjects:

- `feat(ci-go): add swagger drift check`
- `fix(cd-argocd): retry on push conflict`
- `chore(renovate): raise minimumReleaseAge to 3 days`
- `docs: document ci-helm working_directory input`

## Non-Goals for Routine Changes

- Adding a required input or renaming an existing one without migrating every caller.
- Adding a kubeconfig, cluster credential, or Tailscale auth to any workflow.
- Lowering the Trivy severity threshold to get a build green.
- Special-casing a single repo inside `renovate/default.json`.
- Bumping a language version without checking consumer compatibility.
- Large refactors of a workflow without a clear benefit to the repos calling it.
