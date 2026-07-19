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
        "ref": "refs/tags/shared/vendor-broadcast/v1.0.0",
        "default_target_dir": ".github/workflows",
        "dependencies": [
          {
            "shared_repo": "your-org/shared-github-actions",
            "shared_manifest_ref": "main",
            "id": "github-action-common-setup",
            "version": "^1.0.0",
            "default_target_dir": ".github/actions/common-setup"
          }
        ]
      }
    ]
  }
]
```

| Option               | Description                                                  |
| -------------------- | ------------------------------------------------------------ |
| `default_target_dir` | It provides the destination directory inside the consumer repository when a dependency omits `target_dir`. The `target_dir` specified in `gitops-vendor.json` overrides the matched version's `default_target_dir`, so a dependency must configure `target_dir` when its matched shared version has no default. |
| `dependencies`       | It declares dependencies associated with the shared artifact. When a consumer repository subscribes to that shared artifact, those dependencies are automatically subscribed to as well. |

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
   
2. If the `id` exists and the version is new, `gitops-shared.json` is updated idempotently. The new version explicitly inherits `default_target_dir` and `dependencies` from the previous version when they are configured.
3. Then dispatches to all consumer repositories listed in `gitops-subscribers.json`.

## 📥 Consumer Repository Usage

> [!WARNING]
> **Breaking change:** This warning applies only when upgrading from a version earlier than `2.0.0` to version `2.0.0` or later.
>
> Starting with version `2.0.0`, `gitops-vendor-lock.json` uses a new, incompatible format.
>
> Before changing `gitops-vendor.json`, run the upgraded workflow once with the existing dependency configuration. This rebuilds `gitops-vendor-lock.json` in the new format.
>
> The workflow does not trust incompatible lock data. During this rebuild, it cannot infer previous shared repository subscriptions or remove shared artifact paths recorded only in the old lock file.

Use this project on each **consumer repository** to synchronize shared artifacts by id with a SemVer range.

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
            "version": "^1.0.0"
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
