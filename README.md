# Macro Deck GitHub Actions

Reusable workflows for publishing to the Macro Deck Store.

| Workflow | Purpose |
| --- | --- |
| [`publish-plugin.yml`](.github/workflows/publish-plugin.yml) | Builds and packs a plugin with the Macro Deck plugin CLI and uploads it to the Platform's build library |

## Publishing a plugin

Add this to the plugin repository as `.github/workflows/release.yml`:

```yaml
name: Release

on:
  release:
    types: [published]

jobs:
  publish:
    uses: Macro-Deck-App/GitHub-Actions/.github/workflows/publish-plugin.yml@v1
    permissions:
      contents: read
      id-token: write
    with:
      package-id: com.example.my-plugin
      version: ${{ github.event.release.tag_name }}
      source: src/MyPlugin
      changelog: ${{ github.event.release.body }}
```

Publishing a release on GitHub then builds the plugin, and the upload waits in the
project's build library in the Creator Portal, where a release is started from it.

- No secret is involved. The job authenticates with a GitHub Actions OIDC token; the
  Platform reads repository, commit, tag and run from it and accepts it only from this
  workflow at a `v*` tag, and only for the repository connected to the project.
- The release's tag is the version (one leading `v` is dropped): the workflow writes it into
  the plugin's `manifest.json` and builds with it as the assembly version. The build number
  defaults to the run number.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `package-id` | yes | The package identifier registered in the Creator Portal. |
| `version` | yes | The version the build declares; a leading `v` is dropped. |
| `source` | yes | The plugin project directory, holding `manifest.json` and `macrodeck-build.json`. |
| `build` | no | Build identifier; defaults to the run number. |
| `changelog` | no | Becomes the default changelog of a release started from the build. |
| `cli-version` | no | `MacroDeck.Plugin.Cli` version; defaults to the newest prerelease. |
| `platform-url` | no | Defaults to `https://api.macro-deck.app`. |
| `audience` | no | Defaults to `https://api.macro-deck.app`. |

## Releasing this repository

The Platform trusts this workflow only at a tag matching `v*`. Tag a release (and move
the major tag, for example `v1`, that callers reference) after changing it.
