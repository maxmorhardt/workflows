# Renovate

Shared dependency-update preset for every maxmorhardt repo. Each repo has a two-line `renovate.json` extending [default.json](default.json), so policy changes happen here once.

## What it does

- **Weekly batch** (Mondays before 6am ET): all minor/patch updates grouped into one PR per repo, **automerged** once CI passes. Majors get individual PRs and wait for manual review.
- **Security fixes bypass the schedule**: vulnerability PRs (labeled `security`) open immediately.
- **Supply-chain guard**: `minimumReleaseAge: 3 days` means brand-new package releases are not picked up until they have aged.
- **Lockfile maintenance** monthly: refreshes transitive pins.
- **Go hygiene**: `gomodTidy` runs after every go.mod update.
- Covers npm/pnpm, Go modules, Dockerfiles (base images), and GitHub Actions across all repos.
- A **Dependency Dashboard** issue is created in each repo listing every pending/ignored update in one place.

## One-time setup (manual)

1. Install the Mend Renovate GitHub App: https://github.com/apps/renovate - grant it all repos (or the nine with configs).
2. Enable **Dependabot alerts** in each repo (Settings → Security → Dependabot alerts) — Renovate's `vulnerabilityAlerts` reads these to open immediate security PRs. `osvVulnerabilityAlerts` works without it as a fallback.
3. Merge the `renovate.json` in each repo. Renovate opens an onboarding PR per repo on first run - merge those.

## Notes

- To pause a repo, add `"enabled": false` to its `renovate.json`; to skip one dependency, add a `packageRules` entry there rather than editing the shared preset.
