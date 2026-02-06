# aztec-ci-actions

Reusable GitHub Actions and workflows for Wonderland's Aztec submodule repos.

## Structure

```
actions/
├── setup-aztec/action.yml       # Node, Foundry, Aztec CLI, compile, codegen, PXE
├── benchmark/action.yml         # Download baseline, diff, comment, upload
├── js-tests/action.yml          # yarn test:js wrapper
└── noir-tests/action.yml        # aztec test wrapper

.github/workflows/
├── pr-checks.yml                # Reusable: benchmark + js-tests + noir-tests
├── main-tests.yml               # Reusable: js-tests + noir-tests
└── update-baseline.yml          # Reusable: benchmark baseline management
```

## Usage in submodule repos

### PR checks

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks
on: [pull_request]
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true
jobs:
  checks:
    uses: defi-wonderland/aztec-ci-actions/.github/workflows/pr-checks.yml@main
    secrets: inherit
```

### Main branch tests

```yaml
# .github/workflows/main-tests.yml
name: Main Tests
on:
  push:
    branches: [main]
jobs:
  tests:
    uses: defi-wonderland/aztec-ci-actions/.github/workflows/main-tests.yml@main
    secrets: inherit
```

### Benchmark baseline

```yaml
# .github/workflows/update-baseline.yml
name: Update Benchmark Baseline
on:
  push:
    branches: [dev, main]
  schedule:
    - cron: "0 0 1 * *"
  workflow_dispatch:
jobs:
  baseline:
    uses: defi-wonderland/aztec-ci-actions/.github/workflows/update-baseline.yml@main
    secrets: inherit
```

### Setup action only (for custom workflows)

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: defi-wonderland/aztec-ci-actions/actions/setup-aztec@main
    with:
      start-pxe: "false"
      run-codegen: "false"
```

## What each action does

### `setup-aztec`

Full Aztec development environment setup:

1. Node.js 22.17.0
2. Add `~/.aztec` to PATH
3. Detect version from `config.aztecVersion` in `package.json`
4. Install Foundry
5. Install Aztec CLI at detected version
6. (optional) Start local network
7. `yarn --frozen-lockfile`
8. `aztec compile`
9. (optional) `aztec codegen`
10. (optional) Start PXE node

### `benchmark`

Benchmark comparison on PRs:

1. Download baseline artifact from base branch
2. Generate markdown diff via `aztec-benchmark/action`
3. Comment diff on PR
4. Upload new baseline as artifact

### `js-tests` / `noir-tests`

Thin wrappers around `yarn test:js` and `aztec test` with proper env vars and terminal allocation.

## Versioning

Pin to `@main` during development. Tag releases as `v1`, `v1.1`, etc. once stable. Submodules pin to `@v1` (major) for stability.
