# .releaserc.json Templates

## How the config is composed

The `.releaserc.json` has two variable parts:
1. **`branches`** — always a single release branch
2. **`plugins`** array — depends on whether ZIP assets are enabled

Everything else is constant. Compose the final config by picking the right plugins combination.

## Branches

Always a single release branch:
```json
"branches": ["{{MAIN_BRANCH}}"]
```

> **No beta branch in semantic-release.** Beta testing is handled separately — either locally via `beta.sh` or via a manual GitHub Actions trigger (`workflow_dispatch`). This avoids merge conflicts between diverging branches.

## Plugins: without ZIP assets

When there's no ZIP, the `exec` plugin updates `style.css` directly via `sed`, and `@semantic-release/git` commits the changed files back.

```json
"plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    [
        "@semantic-release/changelog",
        {
            "changelogFile": "CHANGELOG.md"
        }
    ],
    [
        "@semantic-release/exec",
        {
            "prepareCmd": "sed -i 's/^Version: .*/Version: ${nextRelease.version}/' style.css"
        }
    ],
    [
        "@semantic-release/git",
        {
            "assets": [
                "CHANGELOG.md",
                "style.css",
                "package.json",
                "package-lock.json"
            ],
            "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
        }
    ],
    "@semantic-release/github"
]
```

## Plugins: with ZIP assets

When ZIP assets are enabled, the `exec` plugin calls `build-release.sh` which handles both the `style.css` update and ZIP creation. The `@semantic-release/github` plugin then attaches the ZIP to the GitHub release.

```json
"plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    [
        "@semantic-release/changelog",
        {
            "changelogFile": "CHANGELOG.md"
        }
    ],
    [
        "@semantic-release/exec",
        {
            "prepareCmd": "bash build-release.sh ${nextRelease.version}"
        }
    ],
    [
        "@semantic-release/git",
        {
            "assets": [
                "CHANGELOG.md",
                "style.css",
                "package.json",
                "package-lock.json"
            ],
            "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
        }
    ],
    [
        "@semantic-release/github",
        {
            "assets": [
                {
                    "path": "{{THEME_SLUG}}.zip",
                    "label": "{{THEME_SLUG}}.zip"
                }
            ]
        }
    ]
]
```

## Important notes

- **Plugin order matters.** `commit-analyzer` and `release-notes-generator` must come first. `git` must come before `github` so that the version commit is included in the release.
- **`[skip ci]`** in the git commit message prevents the release commit from triggering another workflow run.
- **`sed -i`** uses GNU sed syntax (no `''` after `-i`). This is correct for GitHub Actions (Ubuntu). It does NOT work on macOS — but it only runs in CI, never locally.
- The `${nextRelease.version}` and `${nextRelease.notes}` are semantic-release template variables, not shell variables.
