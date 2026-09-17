# Macro Deck GitHub Actions

Reusable workflows and actions for publishing to the Macro Deck Store.

| Workflow / action | Purpose |
| --- | --- |
| [`publish-plugin.yml`](.github/workflows/publish-plugin.yml) | Builds and packs a plugin with the Macro Deck plugin CLI and uploads it to the Platform's build library |
| [`actions/setup-plugin-cli`](actions/setup-plugin-cli/action.yml) | Installs .NET and the `MacroDeck.Plugin.Cli` tool |
| [`actions/build-plugin`](actions/build-plugin/action.yml) | Writes the version into `manifest.json` and builds a `.macroDeckPlugin` |

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
- The package identifier is the `id` in `manifest.json`; it has to match the package
  registered in the Creator Portal.
- The release's tag is the version (one leading `v` is dropped): the workflow writes it into
  the plugin's `manifest.json` and builds with it as the assembly version. The build number
  defaults to the run number.

## Dependency list

After building, the workflow lists the NuGet packages the plugin project restored and uploads them
with the build as `dependencies.json`. The Creator Portal shows them per build, on the version started
from it and to the moderator reviewing it, and the Platform's security scan reads them. Nothing has to
be configured.

It is optional and never fails a release: if the project cannot be found or listed, the file is left
out with a warning and the build is uploaded without it.

- **The project** is the one `macrodeck-build.json` publishes (the first `*.csproj`, `*.fsproj` or
  `*.vbproj` argument of a `dotnet` target), or else the only project file in `source`.
- **Packages** come from `dotnet list package --include-transitive --format json`, per target
  framework, so central package management and multi-targeting are covered.
- **Vulnerabilities** come from a second call with `--vulnerable`, which needs the feeds' vulnerability
  data (nuget.org's). If that call fails, the list is uploaded with `vulnerabilitiesChecked: false`. A
  feed that publishes no vulnerability data reports none.
- **Feeds** are the configured restore sources (`project.restore.sources` of the assets file). Each
  package's own feed is read from the `.nupkg.metadata` NuGet writes into the packages folder. A folder
  inside the checkout is written relative to it (`./local-feed`); a URL loses any credentials and query
  string. No secret reaches the file.
- **Limits.** The Platform refuses a list over 1 MiB or with more than 5,000 package references, so the
  workflow leaves such a list out rather than fail the upload.

```json
{
  "schemaVersion": 1,
  "generatedBy": "Macro-Deck-App/GitHub-Actions publish-plugin.yml",
  "sdkVersion": "10.0.101",
  "vulnerabilitiesChecked": true,
  "sources": ["./local-feed", "https://api.nuget.org/v3/index.json"],
  "frameworks": [
    {
      "framework": "net10.0",
      "packages": [
        {
          "id": "Newtonsoft.Json",
          "requestedVersion": "12.0.1",
          "resolvedVersion": "12.0.1",
          "direct": true,
          "source": "https://api.nuget.org/v3/index.json",
          "vulnerabilities": [
            { "severity": "High", "advisoryUrl": "https://github.com/advisories/GHSA-5crp-9r3c-p9vr" }
          ]
        },
        {
          "id": "Microsoft.Extensions.Logging.Abstractions",
          "resolvedVersion": "8.0.0",
          "direct": false,
          "source": "https://api.nuget.org/v3/index.json",
          "vulnerabilities": []
        }
      ]
    }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `schemaVersion` | `1`. A breaking change to the format gets a new number |
| `generatedBy`, `sdkVersion` | Which workflow and .NET SDK wrote it |
| `vulnerabilitiesChecked` | Whether the vulnerability check ran |
| `sources` | The configured restore feeds |
| `frameworks[].framework` | A target framework of the project |
| `packages[].id`, `resolvedVersion` | The package and the version restore resolved |
| `packages[].requestedVersion` | What the project asked for; direct dependencies only |
| `packages[].direct` | Referenced by the project itself rather than by another package |
| `packages[].source` | The feed it came from: a URL, `./path` inside the repository, or an absolute path. Omitted when unknown |
| `packages[].vulnerabilities` | `severity` (`Low`, `Moderate`, `High`, `Critical`) and `advisoryUrl` |

The list is trusted exactly as much as the build: it is written by this workflow, in the same job
the plugin's own MSBuild code runs in. It helps a moderator; it is not a verified bill of materials.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `version` | yes | The version the build declares; a leading `v` is dropped. Written into `manifest.json`. |
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
| `build-plugin` | `source` (required), `version` (written into `manifest.json` when set), `output-directory` | `package-path`, `package-file-name` |

There is no upload action: the Platform accepts builds only from `publish-plugin.yml`, so
publishing always goes through the reusable workflow.

## Releasing this repository

The Platform trusts the workflow only at a tag matching `v*`. The workflow runs the actions
at the major tag (`@v1`), so tag a release and move the major tag, for example `v1`, after
changing the workflow or an action. A new major version also has to update the `@v1`
references inside `publish-plugin.yml`.
