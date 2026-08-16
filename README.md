# gha-workflows

Collection of Reusable Workflows.

## Available workflows

| Workflow | Purpose |
| --- | --- |
| [`container-image.yml`](.github/workflows/container-image.yml) | Builds a container image with Podman via `make container-build` and publishes it to the registry. |
| [`github-release.yml`](.github/workflows/github-release.yml) | Publishes a GitHub Release with `softprops/action-gh-release` and attaches the distribution assets. |

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
