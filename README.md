# studio-ci

Shared CI building blocks for the [igonzalezespi](https://github.com/igonzalezespi) studio,
consumed by **pinned git reference** the same way as
[`@studio/eslint-config`](https://github.com/igonzalezespi/eslint-config) and
[`@studio/tsconfig`](https://github.com/igonzalezespi/tsconfig) — so every project's CI shares one
source of truth instead of drifting per-repo copies.

Public because the logic isn't secret (no tokens, IPs, or anything sensitive), and a public
action resolves everywhere with zero setup.

## Actions

### `ci-gate`

One aggregate gate per workflow: the single required check. Call it from a final job that depends
on every other job, so path/label-skipped jobs never wedge the merge — `skipped` and `success`
both pass; only `failure`/`cancelled` fail it.

```yaml
jobs:
  # ... your jobs ...
  ci-gate:
    name: ci-gate
    if: always()
    needs: [lint, typecheck, test, build] # list EVERY job
    runs-on: ubuntu-latest
    steps:
      - uses: igonzalezespi/studio-ci/ci-gate@v0.1.2
        with:
          needs-json: ${{ toJSON(needs) }}
```

### `detect-changes`

Classify a PR's changed files into canonical buckets so each job runs only when relevant — a
docs-only PR runs only doc checks, and e2e/heavy jobs skip when no functional code changed.
Bi-stack defaults (JS/TS monorepos and Flutter); a repo overrides only the roots that differ via
the `filters` input.

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read # REQUIRED: paths-filter lists PR files via the API on pull_request
    outputs:
      functional: ${{ steps.d.outputs.functional }}
      e2e_relevant: ${{ steps.d.outputs.e2e_relevant }}
      docs: ${{ steps.d.outputs.docs }}
    steps:
      - uses: actions/checkout@v4
      - id: d
        uses: igonzalezespi/studio-ci/detect-changes@v0.1.2

  e2e:
    needs: changes
    if: needs.changes.outputs.e2e_relevant == 'true'
    runs-on: ubuntu-latest
    steps: [...]
```

> **The `changes` job needs `pull-requests: read`.** On `pull_request`, paths-filter lists the
> changed files through the GitHub API, which requires that scope. If your workflow sets a
> top-level `permissions:` block (e.g. `contents: read`), that becomes the job's *full* grant and
> silently drops `pull-requests` — so set the permission **on the `changes` job** as shown above,
> not only at the top level. Symptom when it's missing: `Resource not accessible by integration`.

Buckets: `docs`, `ci`, `deps`, `code`, `e2e_relevant`, `db_migration`, `i18n`, `assets`, plus a
derived `functional` — true for anything but a **pure docs change**. A `ci` or `deps` change counts
as functional on purpose (fail-safe: a workflow or lockfile change can break the build, so run the
full suite). Override the defaults:

```yaml
      - uses: igonzalezespi/studio-ci/detect-changes@v0.1.2
        with:
          filters: |
            e2e_relevant:
              - 'apps/web/**'
```

### `coverage-stale-gate`

Throttle an **activity-driven** coverage workflow so it runs **at most once per window** without a
cron. Built for self-hosted desktop runners that are off at night: instead of a nightly `schedule:`
(which would queue indefinitely while the machine is off), trigger coverage on `push` to your
integration branch and let this action skip it unless the last successful run is stale. The state is
the workflow's own run history (queried via the API) — no cache, no artifact, no committed timestamp.

```yaml
on:
  push:
    branches: [develop]
  workflow_dispatch:

permissions:
  contents: read
  actions: read                 # coverage-stale-gate reads this repo's run history

jobs:
  gate:
    runs-on: ubuntu-latest
    outputs:
      stale: ${{ steps.gate.outputs.stale }}
    steps:
      - id: gate
        uses: igonzalezespi/studio-ci/coverage-stale-gate@v0.2.0
        with:
          workflow: coverage.yml   # this file
          branch: develop
          max-age-days: "7"

  coverage:
    needs: gate
    if: needs.gate.outputs.stale == 'true'
    runs-on: ubuntu-latest
    steps: [...]                  # run the real coverage + upload here
```

> **Needs `actions: read`** on the gate job (run-history read) and `gh` on the runner (preinstalled
> on github-hosted and on the studio runner image). Keep this workflow **out of any required check** /
> `ci-gate.needs` so a coverage run never blocks a PR. Edge cases: never-run → stale (bootstraps on
> the first push); a week with no push → no run (the last number is still valid); bursts of pushes →
> only the first past the window runs (the rest see a fresh success and skip).

## Versioning

Tagged `vMAJOR.MINOR.PATCH`; consumers pin a tag (`@v0.1.2`) and Renovate bumps the ref. Third-party
actions inside are SHA-pinned.

## License

MIT — see [`LICENSE`](./LICENSE).
