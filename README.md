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
      - uses: igonzalezespi/studio-ci/ci-gate@v0.1.0
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
    outputs:
      functional: ${{ steps.d.outputs.functional }}
      e2e_relevant: ${{ steps.d.outputs.e2e_relevant }}
      docs: ${{ steps.d.outputs.docs }}
    steps:
      - uses: actions/checkout@v4
      - id: d
        uses: igonzalezespi/studio-ci/detect-changes@v0.1.0

  e2e:
    needs: changes
    if: needs.changes.outputs.e2e_relevant == 'true'
    runs-on: ubuntu-latest
    steps: [...]
```

Buckets: `docs`, `ci`, `deps`, `code`, `e2e_relevant`, `db_migration`, `i18n`, `assets`, plus a
derived `functional` (true when any non-`docs`/`ci`-only bucket matched). Override the defaults:

```yaml
      - uses: igonzalezespi/studio-ci/detect-changes@v0.1.0
        with:
          filters: |
            e2e_relevant:
              - 'apps/web/**'
```

## Versioning

Tagged `vMAJOR.MINOR.PATCH`; consumers pin a tag (`@v0.1.0`) and Renovate bumps the ref. Third-party
actions inside are SHA-pinned.

## License

MIT — see [`LICENSE`](./LICENSE).
