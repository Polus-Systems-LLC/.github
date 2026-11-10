# Polus-Systems-LLC/.github

Organization defaults for Polus Systems. This repository must stay public:
GitHub only serves default community health files from a public `.github`
repository, and private repositories can call its reusable workflows.

## Contents

- `SECURITY.md`: default security policy for every repository without its own.
- `.github/pull_request_template.md`: default PR checklist.
- `.github/workflows/naming.yml`, `gitleaks.yml`, `semgrep.yml`: reusable PR
  checks required by org ruleset A (managed in `polus-infra`).
- `.github/workflows/pr-naming.yml`, `pr-scans.yml`: this repository's own
  callers, which double as the reference recipe below.

## Using the checks in a repository

Add two workflow files. The job ids form the required check names
(`naming / check`, `gitleaks / scan`, `semgrep / scan`), so keep them exactly.

`.github/workflows/pr-naming.yml`:

```yaml
name: pr-naming
on:
  pull_request:
    types: [opened, edited, reopened, synchronize]
  merge_group:
permissions: {}
jobs:
  naming:
    permissions: {}
    uses: Polus-Systems-LLC/.github/.github/workflows/naming.yml@<sha> # vX.Y.Z
```

`.github/workflows/pr-scans.yml`:

```yaml
name: pr-scans
on:
  pull_request:
    types: [opened, edited, reopened, synchronize]
  merge_group:
permissions: {}
concurrency:
  group: pr-scans-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
jobs:
  gitleaks:
    permissions:
      contents: read
    uses: Polus-Systems-LLC/.github/.github/workflows/gitleaks.yml@<sha> # vX.Y.Z
  semgrep:
    permissions:
      contents: read
    uses: Polus-Systems-LLC/.github/.github/workflows/semgrep.yml@<sha> # vX.Y.Z
```

Then add the repository to ruleset A in `polus-infra`.

Rules for callers:

- No `if:` and no `name:` on the caller jobs, and no `paths` or `branches`
  filters on the triggers. A skipped job reports success to a required check,
  and a workflow that never starts leaves the check waiting forever.
- Both callers include `edited`: GitHub sends it when the PR's title, body
  or base branch changes, and the scans trust the base branch, so a
  retargeted PR must be scanned again. Keep `cancel-in-progress` in
  `pr-scans.yml` and never add it to `pr-naming.yml`.
- Pin a full commit SHA that is on this repository's `main`. Before approving
  any pin change, run
  `gh api repos/Polus-Systems-LLC/.github/compare/<sha>...main --jq .status`;
  it must print `ahead` or `identical`. Anything else (for example a commit
  that exists only in a fork) is rejected.
- Code-own the files that control the checks: `/.github/`, `.gitattributes`,
  `**/.gitattributes`, `.gitignore`, `.gitleaks.toml`, `.gitleaksignore`,
  `.semgrepignore` and `**/.semgrepignore`.

## Clearing a false positive

- Gitleaks reads `.gitleaks.toml` and `.gitleaksignore` from the PR's base
  commit, never from the PR, and ignores `gitleaks:allow` comments. Copy the
  finding's `Fingerprint` from the scan log into `.gitleaksignore` in a
  separate, code-owner-reviewed PR. It takes effect once merged; then update
  the blocked PR from its base branch. The commit-less form
  `<file>:<rule>:<line>` survives a rebase. A base `.gitleaks.toml` must not
  use a relative `[extend] path`: the scan runs outside the checkout, so the
  path does not resolve and the scan fails.
- Semgrep reports only findings the PR introduces. Suppress one with a
  `nosemgrep` comment on the line; each added suppression is annotated as a
  warning for the reviewer. `.semgrepignore` files are also read from the
  base commit, so a change to them takes effect once merged. They must not
  use `:include`: it would read a file such as `.gitignore` from the PR, so
  the scan fails instead. List the patterns directly.

## Naming convention

- Branch: `<type>/PLS-<number>-<slug>`, type one of feat, fix, hotfix, chore,
  docs, refactor, test, ci; slug in lowercase letters, digits and hyphens.
- PR title: `PLS-<number>: <summary>`.
- PRs opened by bots (Dependabot and other apps) are exempt, and GitHub's
  Revert button output (`revert-<N>-...` branch, `Revert "..."` title) passes.
- To fix a failing title, edit the PR title; that re-runs the check. Re-running
  the failed job instead re-reads the original event's title and fails again.
  A non-conforming branch needs a new PR from a correctly named branch.

## Merge queue

Both callers trigger on `merge_group`, so the required checks report there:
naming passes (the PR was already checked) and the scans cover the merge
group's base to head.

## Emergency path

Ruleset A has no bypass. If the checks block an urgent fix (for example a
Semgrep registry or Docker Hub outage), an org owner temporarily sets ruleset
A to `disabled` in the organization settings UI (the change is audit-logged),
merges, sets it back to active, then reconciles the state through
`polus-infra`.

## Versioning

Releases are tagged `vX.Y.Z` on `main`. Callers pin `@<sha> # vX.Y.Z` so
Dependabot can propose bumps. The Gitleaks version and checksum and the
Semgrep image digest live in `env` strings that Dependabot does not track;
bump them by hand.
