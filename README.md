# fullstackhouse/.github

Shared GitHub Actions for Full Stack House repos. Public so any repo, public or private and
in any org, can call it. It holds workflow code only: no secrets, nothing client-specific.

Pin every reference to a commit SHA; Dependabot's `github-actions` ecosystem keeps it current.

## Security gate: `.github/workflows/security-scan.yml`

Runs on PRs and pushes. It looks only at what the change adds, so old findings never block.

- **gitleaks** scans the new commits for secrets. A hit means rotate the secret, not just
  delete it. False positive: add a `gitleaks:allow` comment on the line.
- **Semgrep CE** (`p/typescript p/nodejs p/secrets`) reports findings new against the base
  commit. False positive: add a `nosemgrep` comment.
- Require one check in the ruleset: **`security / result`**.

```yaml
# .github/workflows/security.yml
name: security
on:
  pull_request:
  push:
    branches: [main]
permissions: {}
concurrency:
  group: security-${{ github.ref }}
  cancel-in-progress: true
jobs:
  security:
    uses: fullstackhouse/.github/.github/workflows/security-scan.yml@<sha> # main
    permissions:
      contents: read
```

## Image scan: `actions/trivy-image`

A step in the job that builds or pushes the image. It fails only on a CRITICAL with a fix
available, and puts the rest in the job summary. Accept a finding in `.trivyignore.yaml`
with `expired_at` and `statement`.

```yaml
- uses: fullstackhouse/.github/actions/trivy-image@<sha> # main
  with:
    image: ghcr.io/fullstackhouse/app:${{ github.sha }}
```

## Scorecard: `.github/workflows/scorecard.yml`

Monthly OpenSSF Scorecard, written to the job summary. Call it from a `schedule` and
`workflow_dispatch` workflow on the default branch (Scorecard refuses other branches), with
`permissions: { contents: read, actions: read, checks: read, issues: read, pull-requests: read }`.

All tools are pinned by checksum or image digest. `trivy-selftest.yml` proves the image scan
fails on `node:18.0.0` and passes on distroless whenever the action changes.
