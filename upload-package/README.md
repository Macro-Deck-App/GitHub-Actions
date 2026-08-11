# Upload Macro Deck package

Packs a built package together with its build metadata and uploads it to the
Macro Deck Platform.

The upload attaches to the version the project is currently preparing in the
Creator Portal, and is refused unless the version numbers match — so a workflow
cannot publish a version nobody asked for. From there the version goes through
review like any other.

## Usage

```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: dotnet build --configuration Release

      - uses: Macro-Deck-App/GitHub-Actions/upload-package@v1
        with:
          token: ${{ secrets.MACRO_DECK_PUBLISH_TOKEN }}
          package-id: com.suchbyte.my-plugin
          artifact: dist/my-plugin.macroDeckPlugin
```

The version defaults to the tag that triggered the run with a leading `v`
removed, so `v1.4.0` publishes `1.4.0`. Pass `version` explicitly when your tags
are named differently.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `token` | yes | Publishing token from the Creator Portal. Pass it from a secret. |
| `package-id` | yes | Reverse-DNS identifier, for example `com.suchbyte.my-plugin`. |
| `artifact` | yes | Path to the built package. |
| `version` | no | Defaults to the triggering tag without its leading `v`. |
| `changelog` | no | Release notes for this version, as CommonMark. |
| `api-url` | no | Defaults to `https://api.macro-deck.app`. |

## Outputs

| Output | Description |
| --- | --- |
| `version` | The version that was uploaded. |
| `size-in-bytes` | Size of the uploaded package. |

## The token

Create one in the Creator Portal under API tokens. A token belongs to a creator
or an organization and reaches that context's projects; narrow it to a single
project when a workflow only ever publishes one.

The secret is shown once. Store it as a repository secret — never inline it in a
workflow file, where it would be readable by anyone who can see the repository.

A token's reach is fixed when it is created. If you need different access,
revoke it and create another.

## What is uploaded

A ZIP holding the built package and a `build-metadata.json` beside it:

```json
{
  "schemaVersion": 1,
  "packageId": "com.suchbyte.my-plugin",
  "version": "1.4.0",
  "artifactFileName": "my-plugin.macroDeckPlugin",
  "commitSha": "0123456789abcdef0123456789abcdef01234567",
  "tagName": "v1.4.0"
}
```

The commit and tag are recorded as provenance. They are what lets anyone check
that a published plugin came from the source it claims to.

## Requirements

The runner needs `jq`, `zip` and `curl`. All three are present on GitHub-hosted
runners; a self-hosted runner may need them installed.
