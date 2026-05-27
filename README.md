![gitops-vendor-sync-workflow](https://socialify.git.ci/leoweyr/gitops-vendor-sync-workflow/image?description=1&font=KoHo&forks=1&issues=1&logo=https%3A%2F%2Fraw.githubusercontent.com%2Fleoweyr%2Fgitops-vendor-sync-workflow%2Frefs%2Fheads%2Fdevelop%2Fassets%2Ficon.svg&name=1&owner=1&pattern=Formal+Invitation&pulls=1&stargazers=1&theme=Light)

![Usage](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fabacus.jasoncameron.dev%2Fget%2Fleoweyr%2Fgitops-vendor-sync-workflow-usage&query=%24.value&label=Usage&color=blue&suffix=%20times)

> [!IMPORTANT]
>
> The `ACCESS_TOKEN` (GitHub Token) secret must have enough permissions for:
>
> 1. Read and write access to the shared code repository.
> 2. Sending `repository_dispatch` events to consumer repositories.

## 📤 Shared Code Repository Usage

Use this project on your **shared code repository** to publish versioned shared artifacts and broadcast update events.

1. Copy `.github/workflows/vendor-broadcast.yml` into your shared code repository.
2. Add `gitops-shared.json` at repository root.
3. Add `gitops-subscribers.json` at repository root to declare consumer repositories.

`gitops-shared.json` example:
```json
[
  {
    "id": "vendor-broadcast",
    "versions": [
      {
        "version": "1.0.0",
        "source": ".github/workflows/vendor-broadcast.yml",
        "ref": "refs/tags/shared/vendor-broadcast/v1.0.0"
      }
    ]
  }
]
```

`gitops-subscribers.json` (consumer repository list) example:
```json
[
  "your-org/consumer-repo-a",
  "your-org/consumer-repo-b"
]
```

Tag-driven publish flow:
1. Create a tag with format `shared/<id>/v<version>`, for example:
   ```bash
   git tag shared/vendor-broadcast/v1.0.0
   git push origin shared/vendor-broadcast/v1.0.0
   ```
   
2. If the `id` exists and the version is new, `gitops-shared.json` is updated idempotently.
3. Then dispatches to all consumer repositories listed in `gitops-subscribers.json`.

## 📥 Consumer Repository Usage

Use this project on each **consumer repository** to pull shared codes by id with SemVer range.

1. Copy `.github/workflows/vendor-sync.yml` into your consumer repository.
2. Add `gitops-vendor.json` at repository root.

`gitops-vendor.json` example:

```json
[
   {
      "shared_repo": "your-org/shared-code-repo-a",
      "shared_manifest_ref": "main",
      "dependencies": [
         {
            "id": "vendor-broadcast",
            "version": "^1.0.0",
            "target_dir": ".github/workflows"
         }
      ]
   },
   {
      "shared_repo": "your-org/shared-code-repo-b",
      "shared_manifest_ref": "main",
      "dependencies": [
         {
            "id": "vendor-sync",
            "version": "^1.0.0",
            "target_dir": ".github/workflows"
         }
      ]
   }
]
```

`version` uses semantic-version range matching rules:

- `1.2.3`: Only match exactly `1.2.3`.
- `~1.2.3`: Allow patch updates within `1.2.x` (for example `1.2.4`), but do not cross to `1.3.0`.
- `^1.2.3`: Allow minor and patch updates within major `1` (for example `1.5.0`), but do not cross to `2.0.0`.
- `1.2.x`: Allow any patch version under minor `1.2`.
- `1.x`: Allow any minor and patch version under major `1`.
- `1`: Equivalent to `1.x`.
- `1.2`: Equivalent to `1.2.x`.

For each dependency id, the workflow selects the highest available version that satisfies the configured range.

`target_dir` means the destination directory inside the current consumer repository where the shared code file is synchronized.

When `shared_repo` in `gitops-vendor.json` changes, `vendor-sync.yml` automatically synchronizes subscription state for your consumer repository:

- It unsubscribes from the previous shared repository (if one exists).
- It subscribes to the new shared repository.

You only need to update `shared_repo`, subscribe and unsubscribe operations are handled automatically.
