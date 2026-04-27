# mbl-actionhub

> Reusable GitHub Actions workflows for MobileByteLabs Kotlin Multiplatform libraries.

The counterpart to [`mbl-actionhub-publish-library-kmp`](https://github.com/MobileByteLabs/mbl-actionhub-publish-library-kmp) (the composite publish action).

## Workflows

| Workflow | Purpose | Use in |
|----------|---------|--------|
| [`ci-kmp-library.yml`](.github/workflows/ci-kmp-library.yml) | CI: quality + platform tests + full assemble | PR + push |
| [`publish-kmp-library.yml`](.github/workflows/publish-kmp-library.yml) | Publish all modules to Maven Central in parallel | Release / schedule |
| [`pr-check-kmp.yml`](.github/workflows/pr-check-kmp.yml) | Fast PR gate (JVM only, skips iOS by default) | PR |

---

## CI — `ci-kmp-library.yml`

Runs in 4 parallel jobs: detect-changes → quality → platform-tests → build-all.

**Smart detection**: only changed `cmp-*` modules are tested on push/PR. Root config changes trigger full rebuild. Publish gate always builds everything.

```yaml
# .github/workflows/gradle.yml  (in your library repo)
name: CI
on:
  push:
    branches: [development, main]
  pull_request:
    branches: [development, main]
  workflow_call:

jobs:
  ci:
    uses: MobileByteLabs/mbl-actionhub/.github/workflows/ci-kmp-library.yml@main
    with:
      module-pattern: 'cmp-'     # optional, default: cmp-
      java-version: '21'          # optional, default: 21
      run-ios-tests: true         # optional, default: true
```

### Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `module-pattern` | `cmp-` | Module directory prefix to discover |
| `java-version` | `21` | JDK version |
| `java-distribution` | `zulu` | JDK distribution |
| `run-ios-tests` | `true` | Run iOS Simulator tests (macOS runner) |
| `run-linux-tests` | `true` | Run Linux Native tests |
| `run-quality` | `true` | Run Spotless + Detekt |

---

## Publish — `publish-kmp-library.yml`

Discovers all `cmp-*` modules with `mavenPublishing` applied → runs CI gate → publishes all in parallel.

```yaml
# .github/workflows/publish.yml  (in your library repo)
name: Publish
on:
  schedule:
    - cron: '0 0 * * 1'   # every Monday
  workflow_dispatch:
    inputs:
      module:
        description: 'Specific module (empty = all)'
        required: false

jobs:
  publish:
    uses: MobileByteLabs/mbl-actionhub/.github/workflows/publish-kmp-library.yml@main
    with:
      module-pattern: 'cmp-'
      module: ${{ inputs.module || '' }}
      java-version: '21'
    secrets: inherit
```

### Required secrets

| Secret | Description |
|--------|-------------|
| `MAVEN_CENTRAL_USERNAME` | Central Portal username |
| `MAVEN_CENTRAL_PASSWORD` | Central Portal password / token |
| `SIGNING_KEY_ID` | GPG key ID (last 8 hex chars) |
| `GPG_KEY_CONTENTS` | Base64-encoded ASCII-armored GPG key |
| `SIGNING_PASSWORD` | GPG passphrase |
| `GH_TOKEN` | GitHub token (optional, for GitHub releases) |

### Encode your GPG key

```bash
gpg --export-secret-keys --armor YOUR_KEY_ID | base64 | pbcopy
```

---

## PR Check — `pr-check-kmp.yml`

Thin wrapper around `ci-kmp-library.yml` — iOS tests disabled by default for speed.

```yaml
# .github/workflows/pr-check.yml
jobs:
  pr-check:
    uses: MobileByteLabs/mbl-actionhub/.github/workflows/pr-check-kmp.yml@main
```

---

## Platform matrix

| Platform | Runner | Tests |
|----------|--------|-------|
| JVM | ubuntu-latest | `jvmTest` |
| iOS Simulator | macos-14 | `iosSimulatorArm64Test` |
| Linux Native | ubuntu-latest | `linuxX64Test` |
| All targets (assemble) | macos-14 | `:module:assemble` |

---

## See also

- [`mbl-actionhub-publish-library-kmp`](https://github.com/MobileByteLabs/mbl-actionhub-publish-library-kmp) — composite publish action
- [`kmp-toolkit`](https://github.com/MobileByteLabs/kmp-toolkit) — reference consumer

## License

Apache 2.0
