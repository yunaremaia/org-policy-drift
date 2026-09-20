# org-policy-drift

> Scan all repos in a GitHub organization and detect repos that diverge from declared standards — missing workflows, outdated actions, absent required files, branch protection gaps.

## The Problem

You have 50 repos in your org. You want them all to have:
- A `README.md`
- A `LICENSE`
- A `.github/workflows/ci.yml`
- Branch protection on `main`
- Dependabot enabled
- A `CODEOWNERS` file

But you have no way to know which repos are missing which requirements. And when you add a new requirement, you have no way to find out which repos need updating.

This is **org policy drift** — the silent gap between your declared standards and your actual repos.

## The Solution

`org-policy-drift` is a read-only, deterministic CLI that:

1. **Declares** your org's policy in a YAML file
2. **Scans** every repo in your org via the GitHub API
3. **Reports** which repos violate which policy rules
4. **Exits non-zero** when drift is detected (CI-ready)

Zero dependencies. No writes. CI-friendly.

## Installation

```bash
pip install git+https://github.com/yunaremaia/org-policy-drift.git
```

## Quick Start

```bash
# Create a policy file
org-policy-drift init

# Scan your org
org-policy-drift scan --org myorg --policy .org-policy.yaml

# Output as JSON for CI
org-policy-drift scan --org myorg --policy .org-policy.yaml --json

# Exit non-zero on drift (for CI gating)
org-policy-drift scan --org myorg --policy .org-policy.yaml --fail-on-drift
```

## Policy File

Create `.org-policy.yaml` in your org's `.github` repo (or any repo):

```yaml
# Org Policy Declaration
# This file defines the standards that all repos in the org must meet.

required_files:
  - README.md
  - LICENSE
  - CODEOWNERS
  - .github/dependabot.yml

required_workflows:
  - ci.yml
  - release.yml

required_branch_protection:
  main:
    required_reviews: 1
    require_code_owner_review: true
    require_status_checks: true
    require_branches_up_to_date: true

required_security:
  dependabot: true
  secret_scanning: true
  code_scanning: true

required_labels:
  - bug
  - enhancement
  - good first issue
  - help wanted

banned_workflows:
  - actions/checkout@v2  # outdated, use v4
  - actions/setup-node@v2  # outdated, use v4

required_topics:
  - myorg
  - oss

# Repos to exclude from policy checks
exclude_repos:
  - myorg/archived-repo
  - myorg/experimental-sandbox

# Per-repo overrides
overrides:
  myorg/special-repo:
    required_workflows: []  # this repo doesn't need workflows
    required_branch_protection: {}
```

## What It Detects

| Drift Type | Severity | Example |
|------------|----------|---------|
| Missing required file | `HIGH` | No `CODEOWNERS` |
| Missing required workflow | `HIGH` | No `ci.yml` |
| Outdated action version | `MEDIUM` | Uses `checkout@v2` instead of `v4` |
| Missing branch protection | `HIGH` | `main` has no protection |
| Missing security feature | `CRITICAL` | Dependabot disabled |
| Missing required label | `LOW` | No `good first issue` label |
| Missing required topic | `LOW` | No `myorg` topic |
| Banned workflow version | `MEDIUM` | Uses deprecated action version |

## CI Integration

### GitHub Actions

```yaml
name: Org Policy Drift Check
on:
  schedule:
    - cron: '0 6 * * *'  # daily at 6am
  workflow_dispatch:

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check org policy drift
        env:
          GITHUB_TOKEN: ${{ secrets.ORG_ADMIN_TOKEN }}
        run: |
          pip install git+https://github.com/yunaremaia/org-policy-drift.git
          org-policy-drift scan --org ${{ github.repository_owner }} --policy .org-policy.yaml --fail-on-drift
```

### GitLab CI

```yaml
org-policy-drift:
  script:
    - pip install git+https://github.com/yunaremaia/org-policy-drift.git
    - org-policy-drift scan --org myorg --policy .org-policy.yaml --fail-on-drift
  only:
    - schedules
```

## Output Formats

### Table (default)

```
REPO                              STATUS    VIOLATIONS
myorg/api-service                 PASS      0
myorg/legacy-app                  FAIL      3
  - Missing CODEOWNERS
  - Missing .github/dependabot.yml
  - Uses actions/checkout@v2
myorg/new-repo                    FAIL      1
  - Missing LICENSE
```

### JSON

```json
{
  "org": "myorg",
  "scanned_at": "2026-09-20T12:00:00Z",
  "total_repos": 50,
  "passing": 42,
  "failing": 8,
  "violations": [
    {
      "repo": "myorg/legacy-app",
      "severity": "HIGH",
      "rule": "required_files",
      "detail": "Missing CODEOWNERS"
    }
  ]
}
```

### SARIF

For GitHub Advanced Security integration:

```bash
org-policy-drift scan --org myorg --policy .org-policy.yaml --sarif > results.sarif
```

## Why Not Just Use a GitHub App?

GitHub Apps like `Mend Renovate` or `Dependabot` handle dependency updates. But they don't:

- Check for missing files across all repos
- Verify branch protection settings
- Detect outdated action versions in workflows
- Enforce labeling conventions
- Gate CI on org-wide compliance

`org-policy-drift` is the missing piece — a single command that tells you exactly which repos need attention.

## Roadmap

- [ ] Auto-fix mode (open PRs to fix drift)
- [ ] Slack/Discord notifications
- [ ] Historical trend tracking
- [ ] Per-team policy overrides
- [ ] Integration with GitHub Advanced Security
- [ ] Pre-commit hook for policy file validation

## License

MIT

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

---

Built with ❤️ by [Yunare Maia](https://github.com/yunaremaia) — open source developer from Mossoró-RN, Brazil.
