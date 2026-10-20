# Renovate

Shared dependency-update preset for every maxmorhardt repo. Each repo has a two-line `renovate.json` extending [default.json](default.json), so policy changes happen here once.

## What it does

- **Rolling batch, no schedule**: all minor/patch updates are grouped into one rolling PR per repo that updates whenever a new update is found (no fixed day/window). Majors get individual PRs.
- **Releases follow what changed**: runtime dependency and base-image bumps are `fix(deps):`, so release-please cuts a patch release and they reach prod; dev dependencies and tooling are `chore(deps):` (no release). Vulnerability PRs (labeled `security`) open immediately as `fix(deps):`.
- **Supply-chain guard**: `minimumReleaseAge: 3 days` means brand-new package releases are not picked up until they have aged.
- **Lockfile maintenance** monthly: refreshes transitive pins.
- **Go**: `gomodTidy` runs after every go.mod update, and the `go` directive is bumped and grouped with the `golang` base image in one `go` PR.
- **Pinned actions**: third-party GitHub Actions are pinned to commit SHAs and kept current; `maxmorhardt/workflows` stays on `@main`.
- Covers npm/pnpm, Go modules, Dockerfiles (base images), and GitHub Actions across all repos.
- A **Dependency Dashboard** issue is created in each repo listing every pending/ignored update in one place.

## One-time setup (manual)

1. Install the Mend Renovate GitHub App: https://github.com/apps/renovate - grant it all repos (or only the repos you want Renovate enabled on).
2. Enable **Dependabot alerts** in each repo (Settings → Security → Dependabot alerts) — Renovate's `vulnerabilityAlerts` reads these to open immediate security PRs. `osvVulnerabilityAlerts` works without it as a fallback.
3. Ensure each repo contains a `renovate.json` (either merge this file directly, or merge Renovate's onboarding PR on first run if it opens one).

## Notes

- To pause a repo, add `"enabled": false` to its `renovate.json`; to skip one dependency, add a `packageRules` entry there rather than editing the shared preset.
