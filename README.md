# .github

[![Supplement Bacon](https://supplement-bacon.com/images/cover2.png)](https://supplement-bacon.com)

Organization-level shared CI/CD repository for **Supplement-Bacon**.\
It centralizes reusable GitHub Actions workflows and composite actions consumed by repositories across the organization.

## Table of Contents

1. [Repository Structure](#repository-structure)
2. [Reusable Workflows](#reusable-workflows)
   1. [Laravel Pint](#laravel-pint)
   2. [PHPUnit](#phpunit)
   3. [SonarQube](#sonarqube)
   4. [Deploy (Forge)](#deploy-forge)
   5. [Frontend Lib CI](#frontend-lib-ci)
3. [Composite Actions](#composite-actions)
   1. [GitHub Token](#github-token)
   2. [PHP](#php)
4. [Usage](#usage)
5. [Required Secrets](#required-secrets)

## Repository Structure

```text
.github/
├── actions/
│   ├── github-token/   # Composite action — generates GitHub App tokens
│   │   └── action.yml
│   └── php/            # Composite action — sets up PHP, Composer & auth
│       └── action.yml
└── workflows/
    ├── forge.yml             # Deploy via Laravel Forge
    ├── frontend-lib-ci.yml   # Format & lint for frontend libraries
    ├── laravel-pint.yml      # PHP code formatting with Laravel Pint
    ├── model-coverage.yml    # Model test coverage reporting
    ├── phpunit.yml           # PHPUnit tests & coverage
    └── sonarqube.yml         # SonarCloud analysis & PR comment
```

## Reusable Workflows

All workflows are triggered via `workflow_call` and are designed to be called from other repositories in the organization.

### Laravel Pint

> `.github/workflows/laravel-pint.yml`

Automatically formats PHP code with [Laravel Pint](https://laravel.com/docs/pint).

| Input                   | Type     | Required | Default | Description                                                                 |
| ----------------------- | -------- | -------- | ------- | --------------------------------------------------------------------------- |
| `repository`            | `string` | ✅       | —       | Fallback repository name when no additional Composer repositories are given |
| `composer_repositories` | `string` | ❌       | `""`    | Newline-separated private repositories needed during Composer install       |
| `extra_owner`           | `string` | ❌       | `""`    | Optional second owner (organization/user) for additional private repos      |
| `extra_repositories`    | `string` | ❌       | `""`    | Newline-separated repositories for `extra_owner`                            |

- Uses the **PHP** composite action to install PHP & Composer.
- On a **pull request**: runs Pint in fix mode and automatically commits formatting fixes via `stefanzweifel/git-auto-commit-action`.
- On a **push**: runs Pint in check mode (`--test`) and fails if formatting is not compliant.

### PHPUnit

> `.github/workflows/phpunit.yml`

Runs PHPUnit tests with coverage handling.

| Input                   | Type      | Required | Default                                   | Description                                                                                |
| ----------------------- | --------- | -------- | ----------------------------------------- | ------------------------------------------------------------------------------------------ |
| `repository`            | `string`  | ✅       | —                                         | Fallback repository name when no additional Composer repositories are given                |
| `composer_repositories` | `string`  | ❌       | `""`                                      | Newline-separated private repositories needed during Composer install                      |
| `test-args`             | `string`  | ❌       | `--parallel --processes=4 --colors=never` | Arguments passed to PHPUnit                                                                |
| `run-example-tests`     | `boolean` | ❌       | `false`                                   | If `true`, skips the standard PHPUnit jobs so example-only test workflows can run instead. |
| `extra_owner`           | `string`  | ❌       | `""`                                      | Optional second owner (organization/user) for additional private repos                     |
| `extra_repositories`    | `string`  | ❌       | `""`                                      | Newline-separated repositories for `extra_owner`                                           |

**2 jobs are defined:**

1. **phpunit** — Runs the full test suite with coverage (`pcov`), then uploads `coverage.xml` as an artifact.
2. **phpunit-coverage** — On pull requests, checks out the base branch and generates a reference coverage report (`coverage-base.xml`) used for SonarQube comparison.

### SonarQube

> `.github/workflows/sonarqube.yml`

Runs [SonarCloud](https://sonarcloud.io/) analysis and posts a coverage comment on pull requests.

| Input        | Type     | Required | Description                                         |
| ------------ | -------- | -------- | --------------------------------------------------- |
| `repository` | `string` | ✅       | Repository name to grant GitHub App token access to |

- Downloads coverage artifacts generated by the **PHPUnit** workflow (PR + base).
- Runs SonarCloud scan via `SonarSource/sonarqube-scan-action`.
- Parses Clover reports to compute coverage diffs (statements, hits, misses, files).
- Fetches SonarCloud quality metrics (bugs, vulnerabilities, code smells, duplications).
- Creates or updates a summary comment on the PR with the full report.

### Deploy (Forge)

> `.github/workflows/forge.yml`

Triggers a deployment through [Laravel Forge](https://forge.laravel.com/).

| Input         | Type     | Required | Description                                         |
| ------------- | -------- | -------- | --------------------------------------------------- |
| `environment` | `string` | ✅       | GitHub environment name (`production` or `staging`) |

**Secrets required:**

| Secret            | Description                   |
| ----------------- | ----------------------------- |
| `FORGE_API_TOKEN` | Laravel Forge API token       |
| `SSH_PRIVATE_KEY` | SSH private key               |
| `SSH_KNOWN_HOSTS` | SSH known hosts configuration |

**Environment Variables:**

Each environment (`production`, `staging`) must define:

| Variable | Description         |
| -------- | ------------------- |
| `DOMAIN` | Forge server domain |

The deployment connects to the server defined in the `DOMAIN` variable of the selected environment.

### Frontend Lib CI

> `.github/workflows/frontend-lib-ci.yml`

Formatting and linting pipeline for frontend libraries (npm).

| Input          | Type     | Default | Description            |
| -------------- | -------- | ------- | ---------------------- |
| `node-version` | `string` | `25`    | Node.js version to use |

- Configures npm to access private `@supplement-bacon` packages via GitHub Packages.
- Installs dependencies (`npm install`).
- Runs formatter (`npm run format`) and auto-commits changes on same-repo PRs.
- Ensures no diff remains after formatting.
- Runs linter (`npm run lint`).

## Composite Actions

### GitHub Token

> `.github/actions/github-token/action.yml`

Generates GitHub App tokens used to access private organization repositories.

| Input          | Required | Default | Description                                                          |
| -------------- | -------- | ------- | -------------------------------------------------------------------- |
| `app_id`       | ✅       | —       | GitHub App ID                                                        |
| `private_key`  | ✅       | —       | GitHub App private key                                               |
| `owner`        | ❌       | `""`    | Owner (organization or user) for token generation                    |
| `repositories` | ❌       | `""`    | Unused (kept for compatibility/documentation); token is owner-scoped |

| Output  | Description                                                                        |
| ------- | ---------------------------------------------------------------------------------- |
| `token` | GitHub App token for the given owner (all repositories where the App is installed) |

### PHP

> `.github/actions/php/action.yml`

All-in-one action that sets up a complete PHP environment.

| Input                | Required | Default                             | Description                                                     |
| -------------------- | -------- | ----------------------------------- | --------------------------------------------------------------- |
| `app_id`             | ✅       | —                                   | GitHub App ID                                                   |
| `private_key`        | ✅       | —                                   | GitHub App private key                                          |
| `repositories`       | ✅       | —                                   | Newline-separated repositories for the current owner auth scope |
| `php-version`        | ❌       | `8.4`                               | PHP version                                                     |
| `extensions`         | ❌       | `mbstring, xml, ctype, iconv, intl` | PHP extensions                                                  |
| `coverage`           | ❌       | `none`                              | Coverage driver: `none`, `pcov`, or `xdebug`                    |
| `extra_owner`        | ❌       | `""`                                | Optional second owner (organization/user)                       |
| `extra_repositories` | ❌       | `""`                                | Newline-separated repositories for `extra_owner`                |

**Steps performed:**

1. Generates tokens via the **GitHub Token** composite action.
2. Installs PHP with the requested extensions and coverage driver (`shivammathur/setup-php`).
3. Caches Composer dependencies.
4. Configures Git authentication for the current owner and, optionally, one extra owner.
5. Installs Composer dependencies.

## Usage

From any repository in the organization, call a reusable workflow:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    uses: Supplement-Bacon/.github/.github/workflows/laravel-pint.yml@main
    secrets: inherit
    with:
      repository: table-booking-membership-api
      composer_repositories: |
        laravel-trivec
        laravel-paginable
        laravel-api-toolkit

  test:
    uses: Supplement-Bacon/.github/.github/workflows/phpunit.yml@main
    secrets: inherit
    with:
      repository: table-booking-membership-api
      composer_repositories: |
        laravel-trivec
        laravel-paginable
        laravel-api-toolkit
      test-args: "--parallel --processes=4 --colors=never"

  sonar:
    needs: test
    uses: Supplement-Bacon/.github/.github/workflows/sonarqube.yml@main
    secrets: inherit
    with:
      repository: table-booking-membership-api

  deploy:
    needs: [lint, test, sonar]
    if: github.event_name == 'push' && (github.ref == 'refs/heads/dev' || github.ref == 'refs/heads/main')
    uses: Supplement-Bacon/.github/.github/workflows/forge.yml@main
    secrets:
      FORGE_API_TOKEN: ${{ secrets.FORGE_API_TOKEN }}
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
      SSH_KNOWN_HOSTS: ${{ secrets.SSH_KNOWN_HOSTS }}
    with:
      environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
```

## Required Secrets

The following secrets must be configured at organization level or in the calling repository:

| Secret                                    | Used by                          | Description                             |
| ----------------------------------------- | -------------------------------- | --------------------------------------- |
| `BACKEND_CI_DEPENDENCIES_APP_ID`          | Laravel Pint, PHPUnit, SonarQube | GitHub App ID used for token generation |
| `BACKEND_CI_DEPENDENCIES_APP_PRIVATE_KEY` | Laravel Pint, PHPUnit, SonarQube | GitHub App private key                  |
| `SONAR_TOKEN`                             | SonarQube                        | SonarCloud authentication token         |
| `FORGE_API_TOKEN`                         | Deploy (Forge)                   | Laravel Forge API token                 |
| `SSH_PRIVATE_KEY`                         | Deploy (Forge)                   | SSH private key for Forge server access |
| `SSH_KNOWN_HOSTS`                         | Deploy (Forge)                   | SSH known hosts configuration           |

> **Environment variables:** The `DOMAIN` variable must be set on each GitHub Environment (`production`, `staging`) in the repository settings — not as a secret.

---

Baked with ❤️ by [Supplément Bacon](https://supplement-bacon.com)
