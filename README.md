# Polus-Systems-LLC/.github

Organization defaults for Polus Systems. This repository must stay public:
GitHub only serves default community health files from a public `.github`
repository, and private repositories can call its reusable workflows.

## Contents

- `SECURITY.md`: default security policy for every repository without its own.
- `.github/pull_request_template.md`: default PR checklist.
- `.github/workflows/naming.yml`, `gitleaks.yml`, `semgrep.yml`: reusable PR
  checks required by org ruleset A (managed in `polus-infra`).

## Using the checks in a repository

Add `.github/workflows/pr-checks.yml` with three jobs whose ids are exactly
`naming`, `gitleaks` and `semgrep`, each calling the reusable workflow pinned
to a full commit SHA of this repository, for example
`uses: Polus-Systems-LLC/.github/.github/workflows/naming.yml@<sha>`. The job
ids form the required check names (`naming / check`, `gitleaks / scan`,
`semgrep / scan`). Then add the repository to ruleset A in `polus-infra`.

Give `gitleaks` and `semgrep` `contents: read` and `naming` no permissions.
Trigger on `pull_request` with types `opened, edited, reopened, synchronize`
so a title edit re-runs the naming check.

## Naming convention

- Branch: `<type>/PLS-<number>-<slug>`, type one of feat, fix, hotfix, chore,
  docs, refactor, test, ci; slug in lowercase letters, digits and hyphens.
- PR title: `PLS-<number>: <summary>`.
- Bot PRs (Dependabot and other apps) are exempt.
