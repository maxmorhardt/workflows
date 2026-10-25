# AGENTS

## Project Context

- Project: Workflows (`github.com/maxmorhardt/workflows`)
- Language: YAML (GitHub Actions reusable workflows), plus JSON for the shared Renovate preset
- Purpose: The shared CI/CD library for every `maxmorhardt` repo. Reusable workflows called with `uses:` from other repositories.
- Target environments: GitHub-hosted `ubuntu-latest` runners. Deployment is GitOps, so the deploy workflow commits to the gitops repo and Argo CD reconciles.
- Related repos: every repo in the org consumes these. `gitops` is the target of `cd-argocd.yml`.

**Consumers pin `@main`, not a tag.** A merge here is live for every repo on its next run.

## Repository Layout

- `.github/workflows/` - the reusable workflows (`on: workflow_call`), plus `ci.yml`, this repo's own PR check: actionlint (with shellcheck on every `run:`) and the Renovate config validator.
  - `ci-go.yml` - `go vet`, golangci-lint, the swagger check when the Makefile has `swag-check`, unit and integration tests with the coverage gate (race on), Trivy filesystem scan. Go version from `module_directory` (default `working_directory`)'s `go.mod`. No binary build: Dockerfiles compile Go in a multi-stage build.
  - `ci-node.yml` - pnpm install, type check, lint, `pnpm audit`, tests with coverage, build, Trivy filesystem scan. Node 24.
  - `ci-python.yml` - pip install from `requirements-dev.txt`, ruff lint (default rules), pytest, Trivy filesystem scan. Python and ruff pinned in the workflow.
  - `ci-docker.yml` - hadolint on the Dockerfile, then one Buildx build to an OCI image layout (provenance, SBOM), Trivy scan of each platform in it failing on HIGH and CRITICAL, then a push of those exact bytes with `latest`.
  - `ci-helm.yml` - validates, then packages and pushes a chart to `oci://ghcr.io/<owner>/charts`, named and versioned from `Chart.yaml` unless `version` overrides it.
  - `validate-helm.yml` - renders a local or remote OCI chart, optional `helm lint --strict`, always strict kubeconform on the output.
  - `validate-manifests.yml` - strict kubeconform over raw manifests (files, dirs, globs) against the default and CRD schema catalogs.
  - `validate-terraform.yml` - `fmt -check`, init, validate, and plan for one root module. `fail_on_diff` turns a non-empty plan into a failure, which is how scheduled drift detection works.
  - `cd-terraform.yml` - plans to a file then applies that exact file, so the apply cannot recompute into something different from what was planned. Human review of the diff happens on the pull request plan, not here.
  - `release-please.yml` - maintains release PRs and tags from conventional commits.
  - `pr-title.yml` - enforces a semantic PR title.
  - `cd-promote.yml` - retags an already-pushed image (dev's `sha-<commit>`) as a release version without rebuilding, waiting for the source tag if dev's build is still running. With `path_filters` instead of `source_tag`, it picks the last commit before the released one that changed those filters' paths in the caller's `.github/path-filters.yml`.
  - `cd-argocd.yml` - GitOps deploy. Bumps `image.tag` and/or the chart version (`targetRevision`, an OCI chart version) in `clusters/<environment>/apps/<app>.yaml` in `maxmorhardt/gitops`, commits, pushes with rebase-and-retry. Prod deploys wait on the `production` environment's approval.
- `renovate/default.json` - the shared Renovate preset every repo extends via a two-line `renovate.json`.
- `renovate/README.md` - what the preset does and the one-time GitHub App setup.

## Core Principles

1. Every change ships to every consumer immediately
   - Treat each edit as a breaking-change review. Adding a required input, removing or renaming an input or secret, or changing a default breaks callers, and a caller passing an undeclared input fails before it runs.
2. Few inputs, callers updated together
   - Add an input only when callers genuinely differ; hardcode what is the same everywhere. A breaking change ships with PRs updating every caller (`maxhq`, `maxhq-private`, `gitops`), merged right after this repo's.
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
- Inputs are `snake_case` (`app_name`, `chart_version`, `image_tag`, `working_directory`). Follow existing names rather than introducing synonyms.
- Reference third-party actions by version tag (`actions/checkout@v7`); Renovate pins them to commit SHAs (`helpers:pinGitHubActionDigests`) and keeps the SHAs current. `maxmorhardt/workflows` stays on `@main`, so a push there is the release.
- Current secrets: `docker_username` and `docker_password` (Docker CI push), `release_please_token` (release-please), `gitops_token` (Argo CD deploy), and `aws_access_key_id` / `aws_secret_access_key` / `cloudflare_api_token` / `repo_admin_token` / `tf_vars` (Terraform). Helm CI and PR Title use the built-in `GITHUB_TOKEN`.
- The Terraform workflows need AWS credentials for every module, not just `aws/`, because all state lives in the same S3 backend.
- Bumping Node 24 is a fleet-wide change. Check that consumers' `package.json` engines agree first. Go follows the module's `go.mod`, which Renovate bumps together with the `golang` base image. Terraform, Node, Python, ruff, and actionlint versions are pinned once in their workflows with a `# renovate:` comment that this repo's `renovate.json` tracks.
- Renovate policy changes go in `renovate/default.json`. Per-repo exceptions belong in that repo's `renovate.json` as a `packageRules` entry. Never special-case a single repo in the shared preset.
- Keep `README.md` in sync. It documents each workflow's inputs and is what people read before wiring one up.

## Change Safety Checklist

Before merging a change to any workflow:

- Does every caller still pass only inputs that exist? An undeclared input fails the caller before it runs.
- Did any input, secret, or output get renamed or removed? If so, open the caller PRs in the same round.
- Grep the sibling workspaces for callers and confirm their inputs still line up:
  ```bash
  grep -rn "workflows/.github/workflows/ci-go.yml" ../*/.github/workflows/
  ```
- Does the change add a secret requirement? Every calling repo must have that secret set before merge.
- Does it introduce a credential this repo should not hold?
- Is `README.md` updated to match?

## Testing Guidance

There is no local test harness. A workflow is only truly exercised when a consumer runs it, which is exactly why the safety checklist above matters.

- `ci.yml` runs actionlint and the Renovate validator on every PR; run `actionlint` locally before pushing too.
- Test a risky change on a branch first: point one consumer repo at `@<branch>` temporarily, confirm a green run, then merge here and revert the consumer to `@main`.
- Test risky changes from a throwaway branch in a consumer repo that points at `@<branch>` before merging.
- After merging, watch the next run in at least one consuming repo per affected language rather than assuming success.
- For `cd-argocd.yml` changes, confirm the commit it produces in `gitops` is correct before trusting it. A bad bump reconciles straight into the cluster.

## Renovate Preset Behavior

Worth knowing because it shapes commit traffic in every repo:

- Minor and patch updates batch into one rolling PR per repo as `chore(deps):`, which cuts no release.
- Minor, patch, pin, and digest PRs are merged by Renovate itself once the repo's required checks pass (no repo auto-merge setting needed). Anything on a `0.x` version still waits for review, and a repo opts out with `"automerge": false` in its own `renovate.json`.
- Majors get individual PRs.
- Vulnerability alerts open immediately as `fix(deps):`, so release-please cuts a patch release and the fix deploys.
- `minimumReleaseAge: 3 days` guards against fresh-release supply-chain attacks, and third-party actions are pinned to commit SHAs so a moved tag can't change what runs.
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
