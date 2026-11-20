# nowo-tech/ci-actions

Shared GitHub Actions pieces for the nowo-tech repositories. GitHub only (no Packagist). Consumers pin the moving
major tag: `@v1`.

| Piece | Kind | Replaces |
|-------|------|----------|
| `nowo-tech/ci-actions/changes@v1` | composite action | the light-CI change detection copied into every `ci.yml` |
| `.github/workflows/release.yml` | reusable workflow | `release.yml` (GitHub Release from tag message + CHANGELOG) |
| `.github/workflows/sync-releases.yml` | reusable workflow | `sync-releases.yml` (backfill releases from tags) |
| `.github/workflows/coderabbit.yml` | reusable workflow | `coderabbit.yml` |
| `.github/workflows/stale.yml` | reusable workflow | `stale.yml` |

## `changes` (composite action)

Classifies a run as `light` or `full` from the org variable `NOWO_CI_PR_MODE`
(`light` = every PR light, `auto`/empty = deps/docs-only PRs light, `full` = every PR full, anything else = full).
`push`, tags, `workflow_dispatch` and `schedule` are always full. Fail-closed.

It is a composite action on purpose, not a reusable workflow: the `changes` job stays in the caller's `ci.yml`, so
job names, `needs:` and required status contexts do not change.

```yaml
jobs:
  changes:
    name: Change scope (light or full)
    runs-on: ubuntu-latest
    outputs:
      mode: ${{ steps.classify.outputs.mode }}
    steps:
      - name: Classify the change
        id: classify
        uses: nowo-tech/ci-actions/changes@v1
        with:
          mode-variable: ${{ vars.NOWO_CI_PR_MODE }}
```

| Input | Default | Meaning |
|-------|---------|---------|
| `mode-variable` | `''` | value of `vars.NOWO_CI_PR_MODE` |
| `extra-code-globs` | `''` | newline/space separated bash globs (`*` also matches `/`) that always count as code in auto mode |
| `checkout` | `true` | check out the PR merge commit with `fetch-depth: 2` on `pull_request`; set `false` if the job already did |

| Output | Values |
|--------|--------|
| `mode` | `light` or `full` |
| `heavy` | `true` unless `mode` is `light` |

The job summary gets `### CI mode: ...`, the reason and the changed paths.

## Reusable workflows

A reusable workflow cannot hold the trigger or raise permissions: the caller declares `on:`, `permissions:` and
`concurrency:`; the called workflow only holds the jobs. None of them holds a required status check.

### release

The caller keeps `on: push: tags: ['v*']`, so a tag created by `tag_when_green.sh` starts it exactly as before.
Multi-line outputs use a random `openssl rand` delimiter, never a fixed `EOF`.

```yaml
name: Create Release
on:
  push:
    tags: ['v*']
permissions:
  contents: write
jobs:
  release:
    uses: nowo-tech/ci-actions/.github/workflows/release.yml@v1
    # with:
    #   changelog-path: docs/CHANGELOG.md   # default
    #   upgrading-path: docs/UPGRADING.md   # default; '' disables the link rewrite
```

### sync-releases

```yaml
name: Sync releases
on:
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Only report what would change'
        type: boolean
        default: false
  schedule:
    - cron: '0 7 * * 1'
permissions:
  contents: write
concurrency:
  group: sync-releases-${{ github.repository }}
  cancel-in-progress: false
jobs:
  sync-releases:
    uses: nowo-tech/ci-actions/.github/workflows/sync-releases.yml@v1
    with:
      dry-run: ${{ inputs.dry_run || false }}
```

### coderabbit

```yaml
name: CodeRabbit
on:
  pull_request:
    branches: [main, master]
    types: [opened, synchronize, reopened]
  push:
    branches: [main, master]
  workflow_run:
    workflows: [CI]
    types: [completed]
  workflow_dispatch:
permissions:
  contents: read
  pull-requests: write
concurrency:
  group: coderabbit-${{ github.event.pull_request.number || github.event.workflow_run.id || github.sha }}
  cancel-in-progress: true
jobs:
  coderabbit:
    uses: nowo-tech/ci-actions/.github/workflows/coderabbit.yml@v1
    secrets:
      CODERABBIT_API_KEY: ${{ secrets.CODERABBIT_API_KEY }}
```

### stale

```yaml
name: Stale Issues and Pull Requests
on:
  schedule:
    - cron: '30 1 * * *'
permissions:
  issues: write
  pull-requests: write
jobs:
  stale:
    uses: nowo-tech/ci-actions/.github/workflows/stale.yml@v1
    # with: {days-before-stale: 60, days-before-close: 60}
```

## Versioning

`v1.0.0` is immutable (annotated tag). `v1` is a moving annotated tag that points at the newest `v1.x.y`.
Breaking changes go to `v2`.

## License

MIT, see `LICENSE`.
