# Macro Deck GitHub Actions

Reusable workflows and actions for publishing to the Macro Deck Store.

| Workflow / action | Purpose |
| --- | --- |
| [`publish-plugin.yml`](.github/workflows/publish-plugin.yml) | Builds and packs a plugin with the Macro Deck plugin CLI and uploads it to the Platform's build library |
| [`actions/setup-plugin-cli`](actions/setup-plugin-cli/action.yml) | Installs .NET and the `MacroDeck.Plugin.Cli` tool |
| [`actions/build-plugin`](actions/build-plugin/action.yml) | Checks the version against `manifest.json` and builds a `.macroDeckPlugin` |

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
      # Optional: also keep the package as a workflow artifact.
      upload-artifact: true
      artifact-name: my-plugin
      artifact-retention-days: 14
```

Publishing a release on GitHub then builds the plugin, and the upload waits in the
project's build library in the Creator Portal, where a release is started from it.

- No secret is involved. The job authenticates with a GitHub Actions OIDC token; the
  Platform reads repository, commit, tag and run from it and accepts it only from this
  workflow at a `v*` tag, and only for the repository connected to the project.
- The release's tag must match the version in the plugin's `manifest.json` (one leading
  `v` is dropped). The build number defaults to the run number.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `package-id` | yes | The package identifier registered in the Creator Portal. |
| `version` | yes | The version the build declares; a leading `v` is dropped. |
| `source` | yes | The plugin project directory, holding `manifest.json` and `macrodeck-build.json`. |
| `build` | no | Build identifier; defaults to the run number. |
| `changelog` | no | Becomes the default changelog of a release started from the build. |
| `cli-version` | no | `MacroDeck.Plugin.Cli` version; defaults to the newest prerelease. |
| `upload-artifact` | no | `true` also keeps the `.macroDeckPlugin` as a workflow artifact. Defaults to `false`. |
| `artifact-name` | no | The artifact's name; defaults to the package file name. |
| `artifact-retention-days` | no | Days to keep the artifact (1-90); `0`, the default, uses the repository setting. |
| `platform-url` | no | Defaults to `https://api.macro-deck.app`. |
| `audience` | no | Defaults to `https://api.macro-deck.app`. |

## Using the actions on their own

The workflow's steps are available as composite actions, for example to build a plugin in
CI without publishing it:

```yaml
steps:
  - uses: actions/checkout@v7
  - uses: Macro-Deck-App/GitHub-Actions/actions/setup-plugin-cli@v1
  - id: build
    uses: Macro-Deck-App/GitHub-Actions/actions/build-plugin@v1
    with:
      source: src/MyPlugin
  - uses: actions/upload-artifact@v7
    with:
      name: my-plugin
      path: ${{ steps.build.outputs.package-path }}
```

| Action | Inputs | Outputs |
| --- | --- | --- |
| `setup-plugin-cli` | `cli-version`, `dotnet-version` (default `10.0.x`) | |
| `build-plugin` | `source` (required), `version` (checked against `manifest.json` when set), `output-directory` | `package-path`, `package-file-name` |

There is no upload action: the Platform accepts builds only from `publish-plugin.yml`, so
publishing always goes through the reusable workflow.

## Releasing this repository

The Platform trusts the workflow only at a tag matching `v*`. The workflow runs the actions
at the major tag (`@v1`), so tag a release and move the major tag, for example `v1`, after
changing the workflow or an action. A new major version also has to update the `@v1`
references inside `publish-plugin.yml`.
