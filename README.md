# Macro Deck GitHub Actions

Reusable workflows and actions for publishing to the Macro Deck Store.

| Workflow / action | Purpose |
| --- | --- |
| [`publish-plugin.yml`](.github/workflows/publish-plugin.yml) | Builds a plugin with the Macro Deck plugin CLI, tests it, runs it on a stub host and uploads it to the Platform's build library |
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

## How a publish runs

The workflow runs these jobs, in this order:

| Job | Runner | Permissions | What it does |
| --- | --- | --- | --- |
| `plan` | `ubuntu-latest` | `contents: read` | Reads the manifest's entrypoints and decides which platforms are built, on which runners |
| `build` | `ubuntu-latest`, or one per platform | `contents: read` | Builds the `.macroDeckPlugin`, runs the repository's tests, lists the dependencies |
| `package` | `ubuntu-latest` | none | Only with `build-per-platform`: merges the platform packages into one |
| `stub-host` | one per platform the manifest declares | none | Runs the plugin conformance suite against the package on a disposable stub host on each platform and hands the reports to the upload |
| `upload` | `ubuntu-latest` | `id-token: write` | Packs `build-metadata.json`, the dependency list and the conformance reports, and uploads the build to the Platform |

- **Only the upload job can authenticate.** Every step of a job with `id-token: write` can request
  the OIDC token the Platform accepts, including build scripts, MSBuild targets and tests. So nothing
  from the repository runs in that job: it checks nothing out and only uploads the package the build
  job handed over.
- **The package cannot change on the way.** Its SHA-256 is recorded right after the build, checked
  before the build job hands it over (so the tests cannot have swapped it) and checked again by each
  job that receives it - including `package`, which checks every platform package before merging them
  and records the digest of the merged one.
- **Tests.** `dotnet test -c Release` on `test-path`, or on the repository's only solution (`*.sln`,
  `*.slnx`) at its root. With none or several, the step is skipped with a warning. A failing test
  stops the release.
- **Stub host.** `macrodeck-plugin test --artifact` starts the package the way Macro Deck does and
  checks registration, protocol negotiation, capabilities, timeouts, disconnect and reconnect. The
  full report is the job's summary; only a failed required check stops the release. The plugin runs
  on its own platform, so there is one job for every platform the manifest declares an entrypoint for,
  each on a runner of that platform: `linux-x64` (`ubuntu-latest`), `win-x64` (`windows-latest`),
  `osx-arm64` (`macos-latest`), `linux-arm64` (`ubuntu-24.04-arm`), `win-arm64` (`windows-11-arm`),
  `osx-x64` (`macos-15-intel`), or the runner `runners` names. A Windows entrypoint is run on Windows,
  a macOS one on macOS, a Linux one on Linux. One platform failing does not cancel the others, and any
  of them failing a required check stops the release. A platform without a runner is not run, with a
  warning.

The calling job still grants `id-token: write`, as in the example above: a reusable workflow's jobs
can only narrow the caller's permissions, not add to them.

## Building each platform on its own runner

One Linux runner cross-builds every platform a plain .NET plugin declares, which is the default and
needs nothing. A target that only builds natively - a `net10.0-windows` target, or a native library
compiled per platform - needs `build-per-platform`:

```yaml
    with:
      version: ${{ github.event.release.tag_name }}
      source: src/MyPlugin
      build-per-platform: true
```

`build` then becomes one job per runtime identifier the manifest declares, each building only its own
with [`macrodeck-plugin build --rid`](https://docs.macro-deck.app/cli/build/), and `package` combines
them with [`macrodeck-plugin merge`](https://docs.macro-deck.app/cli/merge/) into the single package
that is uploaded - the one a build on a machine that could build every platform would have produced.
Merging refuses packages that do not belong together: a different plugin or version, a manifest that
differs beyond its `entrypoints`, a runtime identifier twice, or a shared file whose bytes differ.

- The tests and the dependency list run once, on the first platform (Linux when the manifest declares
  it): they are about the repository, not about a runner.
- `upload-artifact` keeps the merged package, not the platform ones.
- The runner per platform is the same table the stub host uses. Override it, or name one for a
  platform not in it, with `runners`: `runners: '{"osx-arm64": "macos-15"}'`.
- One platform failing does not cancel the others, so a run shows every platform that is broken.

## Dependency list

After building, the workflow lists the NuGet packages the plugin project restored and uploads them
with the build as `dependencies.json`. The Creator Portal shows them per build, on the version started
from it and to the moderator reviewing it, and the Platform's security scan reads them. Nothing has to
be configured.

It is required: the Platform refuses a build without it, and checks it against the dependency policy
its System Administrators set - `MacroDeck.*` packages must be official Macro Deck packages and
reach a minimum version, and packages no
plugin may depend on, directly or transitively. If the project cannot be found or listed, the release
fails with the reason, and a refusal by the policy names each offending package.

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

The list is trusted exactly as much as the build: it is written by this workflow, in the job the
plugin's own MSBuild code runs in. It helps a moderator; it is not a verified bill of materials.

## Conformance report

The `stub-host` job runs `macrodeck-plugin test --artifact` against the built package and writes the
report as JSON (the conformance suite's own format, `ConformanceReportWriter.ToJson`), once per
platform. Each is rendered into its job's summary and uploaded with the build as
`conformance/<rid>.json`, for example `conformance/win-x64.json`. The Creator Portal shows them to the
moderator in the review's Conformance tab, one tab per platform, opening on the worst.

It is optional and advisory:

- **A failed required check still stops the release**, as before, so nothing is uploaded.
- **No report** - `run-stub-host: false`, or a manifest that declares no platform a GitHub runner
  provides - uploads the build without one. The review warns that the plugin was never run on a
  stub host.
- **A report the Platform cannot read** never refuses the build; the review shows it as a warning,
  with the reason.
- The review also warns when the plugin never completed the handshake with the stub host, when no
  check passed, when recommended checks failed or checks were inconclusive, and when the report names
  another plugin id or version than the build.

The report comes from the job that ran the plugin, so it is trusted no further than the build. It
tells a moderator what to look at; it decides nothing.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `version` | yes | The version the build declares; a leading `v` is dropped. Written into `manifest.json`. |
| `source` | yes | The plugin project directory, holding `manifest.json` and `macrodeck-build.json`. |
| `build-per-platform` | no | Builds each declared runtime identifier on a runner of its own platform and merges the results. Defaults to `false`. |
| `runners` | no | JSON object naming the runner that builds a runtime identifier, merged over the defaults. Defaults to `{}`. |
| `build` | no | Build identifier; defaults to the run number. |
| `changelog` | no | Becomes the default changelog of a release started from the build. |
| `run-tests` | no | Runs the repository's tests after the build. Defaults to `true`. |
| `test-path` | no | The solution, project or directory `dotnet test` runs; defaults to the only solution at the repository root. |
| `run-stub-host` | no | Runs the conformance suite on a stub host before the upload and uploads its report. Defaults to `true`. |
| `cli-version` | no | `MacroDeck.Plugin.Cli` version; defaults to the newest release carrying every command these actions use (`3.0.0-beta.11`). |
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
| `setup-plugin-cli` | `cli-version` (defaults to `3.0.0-beta.11`), `dotnet-version` (default `10.0.x`) | |
| `build-plugin` | `source` (required), `version` (written into `manifest.json` when set), `rid` (build only that runtime identifier), `output-directory` | `package-path`, `package-file-name` |

There is no upload action: the Platform accepts builds only from `publish-plugin.yml`, so
publishing always goes through the reusable workflow.

## Releasing this repository

The Platform trusts the workflow only at a tag matching `v*`. The workflow runs the actions
at the major tag (`@v1`), so tag a release and move the major tag, for example `v1`, after
changing the workflow or an action. A new major version also has to update the `@v1`
references inside `publish-plugin.yml`.
