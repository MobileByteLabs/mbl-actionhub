# mbl-actionhub

> Reusable GitHub Actions workflows for MobileByteLabs Kotlin Multiplatform libraries.

The counterpart to [`mbl-actionhub-publish-library-kmp`](https://github.com/MobileByteLabs/mbl-actionhub-publish-library-kmp) (the composite publish action).

## Workflows

| Workflow | Purpose | Use in |
|----------|---------|--------|
| [`ci-kmp-library.yml`](.github/workflows/ci-kmp-library.yml) | CI: quality + platform tests + full assemble | PR + push |
| [`publish-kmp-library.yml`](.github/workflows/publish-kmp-library.yml) | Publish all modules to Maven Central in parallel | Release / schedule |
| [`pr-check-kmp.yml`](.github/workflows/pr-check-kmp.yml) | Fast PR gate (JVM only, skips iOS by default) | PR |
| [`release-notes-from-changelog.yml`](.github/workflows/release-notes-from-changelog.yml) | Prepend matching CHANGELOG.md section as "What's changed" above the auto-generated release body | Release event |

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

### Pre-release-aware GitHub Release flag

When `create-github-release: true`, the workflow auto-flags SemVer pre-release tags as pre-releases on the GitHub UI + RSS feed + "Latest release" pointer.

| Resolved version | `prerelease` on the GitHub Release |
|---|---|
| `2.2.0-alpha.0` | `true` (auto-detected) |
| `2.2.0-beta.1` | `true` (auto-detected) |
| `2.2.0-rc.2` | `true` (auto-detected) |
| `2.2.0` | `false` (GA — auto-detected) |
| `2.2.1-snapshot.1` | `false` (unrecognised suffix — auto-detected falls through to GA) |

Override the auto-detection with `force-prerelease`:

```yaml
jobs:
  publish:
    uses: MobileByteLabs/mbl-actionhub/.github/workflows/publish-kmp-library.yml@main
    with:
      create-github-release: true
      force-prerelease: 'auto'   # default — auto-detect via version suffix
      # force-prerelease: 'true'  # force prerelease=true (e.g. unrecognised pre-release suffix)
      # force-prerelease: 'false' # force prerelease=false (re-issued pre-release promoted to GA quality)
    secrets: inherit
```

Pre-release detection mirrors `mbl-actionhub-bump-version` v1.6+'s recognised suffix list: `-alpha.N`, `-beta.N`, `-rc.N` (period-separated counter form). Unrecognised suffixes (`-snapshot.N`, `-pre.N`, custom labels) auto-resolve to `prerelease=false` — override with `force-prerelease: 'true'` if you want them flagged.

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

## Release Notes — `release-notes-from-changelog.yml`

Pairs with `publish-kmp-library.yml`. The publish flow creates releases with only a `## Published Modules` artifact-list block — this workflow runs on `release:created/published/edited` and prepends the matching `## [<version>]` (or `## [Unreleased]` fallback) section from `CHANGELOG.md` as `## What's changed` above the artifact list. Idempotent — re-runs on the same release exit cleanly.

```yaml
# .github/workflows/release-notes.yml  (in your library repo)
name: Release Notes
on:
  release:
    types: [created, published, edited]
  workflow_dispatch:
    inputs:
      tag:
        description: 'Release tag to enrich (e.g. v3.4.0)'
        required: true
        type: string

jobs:
  enrich:
    uses: MobileByteLabs/mbl-actionhub/.github/workflows/release-notes-from-changelog.yml@v1.7.0
    permissions:
      contents: write
    with:
      tag: ${{ github.event.release.tag_name || inputs.tag }}
    secrets: inherit
```

### Inputs

| Input | Default | Description |
|---|---|---|
| `tag` | (required) | Release tag to enrich (e.g. `v3.4.0`) |
| `changelog-path` | `CHANGELOG.md` | Path relative to repo root |
| `heading-label` | `What's changed` | Heading text for the prepended block (no leading `##`) |
| `separator` | `---` | Markdown separator between changelog block and existing body |
| `fail-when-missing` | `false` | Fail the job if no CHANGELOG section is found (default: no-op) |

### Section matching strategy

1. `## [<version>]` (tag with leading `v` stripped) — preferred
2. `## [Unreleased]` — fallback for consumers who haven't curated CHANGELOG into versioned sections yet

The reusable workflow checks out the repo at the release tag — so the CHANGELOG it reads is the one frozen at release time, not whatever's on the default branch HEAD (which may already have a bump-after-release commit).

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
