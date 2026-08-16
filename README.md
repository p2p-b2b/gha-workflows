# gha-workflows

Collection of Reusable Workflows.

## Available workflows

| Workflow | Purpose |
| --- | --- |
| [`release-please.yml`](.github/workflows/release-please.yml) | Automates versioning and releases from Conventional Commits, via [release-please](https://github.com/googleapis/release-please). |
| [`container-image.yml`](.github/workflows/container-image.yml) | Builds a container image with Podman via `make container-build` and publishes it to the registry. |
| [`github-release.yml`](.github/workflows/github-release.yml) | Publishes a GitHub Release with `softprops/action-gh-release` and attaches the distribution assets. |

## Releasing on merge

`release-please.yml` derives versions from commit messages so nobody has to
remember to tag. It is a **two-step** model, and the second step is the one that
releases:

1. A feature PR merges to `main`. release-please opens — or updates — a
   `chore(main): release X.Y.Z` **Release PR** that accumulates every change
   since the last release, writes the `CHANGELOG.md`, and bumps version files.
   Nothing is tagged yet.
2. You merge that Release PR. release-please creates the git tag and the
   GitHub Release.

So a release is still "merge a PR" — it is the *Release PR*. Feature merges queue
into it rather than each cutting a version, which gives you a reviewable
changelog and a release-when-ready gate.

The bump comes from the commit type:

| Commit type | Bump | Example |
| --- | --- | --- |
| `fix:` | patch | `1.2.3` → `1.2.4` |
| `feat:` | minor | `1.2.3` → `1.3.0` |
| `feat!:` or a `BREAKING CHANGE:` footer | major | `1.2.3` → `2.0.0` |
| `chore:`, `docs:`, `refactor:`, `test:`, `ci:` | none | no Release PR |

### ⚠️ Two prerequisites

Both bite silently if missed:

- **Settings → Actions → General → "Allow GitHub Actions to create and approve
  pull requests" must be enabled.** Without it, creating the Release PR fails.
- **A PR opened with the default `GITHUB_TOKEN` does not trigger other
  workflows.** That is GitHub's recursion guard, not a bug here — but it means
  CI will not run on the Release PR. Pass a PAT or GitHub App token as the
  `token` secret if you need checks on it; leave it unset if you don't.

### Simple case — no build artifacts

```yaml
name: Release
on:
  push:
    branches: [main]

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  release:
    uses: slashdevops/gha-workflows/.github/workflows/release-please.yml@v1.0.2
    with:
      release-type: go
```

### Attaching build artifacts to the release

`released` is only `'true'` on the run where the Release PR was merged, so the
build and publish jobs stay skipped on ordinary feature merges.

```yaml
jobs:
  release:
    uses: slashdevops/gha-workflows/.github/workflows/release-please.yml@v1.0.2
    with:
      release-type: go

  build:
    needs: release
    if: needs.release.outputs.released == 'true'
    runs-on: ubuntu-latest
    steps:
      # ... build, then upload an artifact named `dist`

  containers:
    needs: [release, build]
    if: needs.release.outputs.released == 'true'
    uses: slashdevops/gha-workflows/.github/workflows/container-image.yml@v1.0.2

  publish:
    needs: [release, build, containers]
    if: needs.release.outputs.released == 'true'
    uses: slashdevops/gha-workflows/.github/workflows/github-release.yml@v1.0.2
    with:
      tag: ${{ needs.release.outputs.tag-name }}
      generate-release-notes: false # keep release-please's CHANGELOG notes
```

### `release-please.yml` inputs

| Input | Default | Description |
| --- | --- | --- |
| `release-type` | `simple` | `go`, `simple`, `node`, `python`, `ruby`, … Ignored when `config-file` is set. |
| `config-file` | — | Path to `release-please-config.json`. Takes precedence over `release-type`. |
| `manifest-file` | — | Path to `.release-please-manifest.json`. |
| `target-branch` | detected | Branch to open the Release PR against. |
| `path` | — | Release from a subdirectory rather than the repository root. |
| `include-component-in-tag` | `false` | Prefix tags with the component name, for multi-package repos. |
| `release-as` | — | Force a specific version instead of deriving it from commits. |
| `versioning-strategy` | `default` | release-please versioning strategy. |
| `skip-github-release` | `false` | Maintain the Release PR but never tag — useful as a dry run. |
| `skip-github-pull-request` | `false` | Tag straight from `main`, no Release PR. Gives up the reviewable changelog. |

`googleapis/release-please-action` is pinned to `v5.0.0` inside the workflow.
It is not an input because GitHub does not allow expressions in `uses:` — bump it
here and cut a new tag of this repository to roll consumers forward.

Optional secret: `token` — a PAT or GitHub App token, only needed if CI must run
on the Release PR.

### `release-please.yml` outputs

| Output | Description |
| --- | --- |
| `released` | `'true'` when a release was created this run (the Release PR was merged). |
| `pr-created` | `'true'` when a Release PR was opened or updated this run. |
| `tag-name` | Tag created (e.g. `v1.4.0`). Empty unless a release happened. |
| `version` / `major` / `minor` / `patch` | Released version and its components. |
| `sha` | Commit the release was cut from. |
| `html-url` / `upload-url` | Browser and asset-upload URLs of the release. |

## Usage

Pin to a tag rather than a branch so a change here cannot alter a consumer's
release behaviour without an explicit bump.

```yaml
jobs:
  build:
    # ... build and `actions/upload-artifact` with name: dist

  containers:
    needs: build
    uses: slashdevops/gha-workflows/.github/workflows/container-image.yml@v1.0.2

  release:
    needs: [build, containers]
    uses: slashdevops/gha-workflows/.github/workflows/github-release.yml@v1.0.2
    with:
      files: |
        dist/assets/*.zip
```

### `github-release.yml` inputs

| Input | Default | Description |
| --- | --- | --- |
| `artifact-name` | `dist` | Artifact to download, as named by `actions/upload-artifact`. |
| `artifact-path` | `./dist/` | Directory the artifact is extracted into. |
| `files` | `dist/assets/*.zip` | Newline-separated globs to attach to the release. |
| `tag` | the triggering ref | Tag to release. |
| `name` | the tag | Release title. |
| `draft` | `false` | Create as a draft. |
| `prerelease` | `false` | Mark as a prerelease. |
| `make-latest` | `true` | Mark as the repository's latest release. |
| `generate-release-notes` | `true` | Let GitHub generate notes from merged pull requests. |

It outputs `url`, the browser URL of the published release.

Both workflows declare the permissions they need (`contents: write` for the
release, plus `packages: write` and `id-token: write` for the image), so the
calling workflow must grant at least those.
