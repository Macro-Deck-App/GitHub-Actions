# Macro Deck GitHub Actions

Reusable actions for publishing to the Macro Deck Store.

| Action | Purpose |
| --- | --- |
| [`upload-package`](upload-package) | Packs a built package with its build metadata and uploads it to the Platform |

Each action lives in its own directory and is referenced by that path:

```yaml
- uses: Macro-Deck-App/GitHub-Actions/upload-package@v1
```
