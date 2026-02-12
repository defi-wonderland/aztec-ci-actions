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
├── update-baseline.yml          # Reusable: benchmark baseline management
└── pre-release.yml              # Reusable: build + tag + GitHub pre-release
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
    uses: defi-wonderland/aztec-ci-actions/.github/workflows/pr-checks.yml@<tag>
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
    uses: defi-wonderland/aztec-ci-actions/.github/workflows/main-tests.yml@<tag>
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
    uses: defi-wonderland/aztec-ci-actions/.github/workflows/update-baseline.yml@<tag>
    secrets: inherit
```

### Pre-release

```yaml
# .github/workflows/pre-release.yml
name: Pre-Release
on:
  workflow_dispatch:
jobs:
  pre-release:
    uses: defi-wonderland/aztec-ci-actions/.github/workflows/pre-release.yml@<tag>
    secrets: inherit
    permissions:
      contents: write
```

Dependents install the pre-release tarball from the GitHub Release:

```bash
npm install https://github.com/<owner>/<repo>/releases/download/prerelease-<sha>/<tarball-name>.tgz
```

Or in `package.json`:

```json
"<package>": "https://github.com/<owner>/<repo>/releases/download/prerelease-<sha>/<tarball-name>.tgz"
```

### Setup action only (for custom workflows)

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: defi-wonderland/aztec-ci-actions/actions/setup-aztec@<tag>
    with:
      start-pxe: "false"
      run-compile: "true"
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
8. (optional) `aztec compile` — controlled by `run-compile` (default: true)
9. (optional) `aztec codegen` — controlled by `run-codegen` (default: false); requires compile (fails if `run-codegen: true` but `run-compile: false`)
10. (optional) Start PXE node

For **pure-JS repos** (no Noir contracts): set `run-compile: "false"` and `run-codegen: "false"` to skip contract build steps.

### `benchmark`

Benchmark comparison on PRs:

1. Run PR benchmarks (gives the BASE `update-baseline` job time to finish)
2. Download baseline artifact from base branch
3. Generate markdown diff via `aztec-benchmark/action` with `only_report` (comparison only, no re-run)
4. Comment diff on PR
5. Upload new baseline as artifact

### `js-tests` / `noir-tests`

Thin wrappers around `yarn test:js` and `aztec test` with proper env vars and terminal allocation.

## Pre-release workflow

The `pre-release.yml` reusable workflow builds the package and publishes a GitHub pre-release with installable artifacts. This allows dependent repos to test unreleased changes without publishing to npm.

**What it does:**

1. Full Aztec environment setup — `run-compile` (default: true) and `run-codegen` (default: true) control whether to compile Noir contracts and run codegen; set both to `false` for pure-JS repos
2. `yarn build`
3. Generate a pre-release version: `<base-version>-prerelease.<short-sha>`
4. Set the version temporarily (never committed to git)
5. `npm pack` to create an npm-installable `.tgz` tarball
6. `tar -czf dist.tar.gz dist/` for the built output
7. Create a git tag `prerelease-<short-sha>` and a GitHub Release marked as pre-release

**Artifacts attached to each release:**

| Asset | Purpose |
|-------|---------|
| `<name>-<version>.tgz` | npm-installable tarball — use with `npm install <url>` |
| `dist.tar.gz` | Raw `dist/` directory for manual extraction |

Production `npm install` from the npm registry is completely unaffected — pre-releases only exist as GitHub Release assets.

## Versioning

Pin to `@dev` during development. Tag releases as `v1`, `v1.1`, etc. once stable. Submodules pin to `@v1` (major) for stability.
