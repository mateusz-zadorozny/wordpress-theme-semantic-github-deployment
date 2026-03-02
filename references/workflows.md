# GitHub Actions Workflow Templates

## release.yml

This workflow runs semantic-release on every push to the release branch(es).

### Without pre-release:

```yaml
name: Release

on:
  push:
    branches:
      - {{MAIN_BRANCH}}

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "lts/*"

      - name: Install dependencies
        run: npm ci

      - name: Run Semantic Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx semantic-release
```

### With pre-release:

Add the beta branch to the trigger:

```yaml
on:
  push:
    branches:
      - {{MAIN_BRANCH}}
      - {{BETA_BRANCH}}
```

Everything else stays the same. Semantic-release reads `.releaserc.json` to know which branch produces pre-releases.

### Notes on the workflow:

- **`fetch-depth: 0`** — semantic-release needs the full git history to analyze commits and determine the next version.
- **`persist-credentials: false`** — prevents the default GITHUB_TOKEN from being used for git operations. Semantic-release manages its own authentication via the `GITHUB_TOKEN` env var, which lets it push commits (version bump) and create releases.
- **`GITHUB_TOKEN`** — this is the built-in GitHub Actions token. No custom secrets needed. It has write access to contents, issues, and PRs because of the `permissions` block.
- **`npx semantic-release`** — npx runs the locally installed version from devDependencies, ensuring consistent behavior.

---

## pr-lint.yml

This workflow validates that PR titles follow the Conventional Commits format. It blocks merging if the title doesn't match.

### Without pre-release:

```yaml
name: "Lint PR"

on:
  pull_request:
    types:
      - opened
      - edited
      - synchronize
    branches:
      - {{MAIN_BRANCH}}

permissions:
  pull-requests: read

jobs:
  main:
    name: Validate PR title
    runs-on: ubuntu-latest
    steps:
      - uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            docs
            style
            refactor
            perf
            test
            build
            ci
            chore
            revert
          requireScope: false
```

### With pre-release:

Add the beta branch:

```yaml
    branches:
      - {{MAIN_BRANCH}}
      - {{BETA_BRANCH}}
```

### Notes on PR lint:

- **`synchronize`** event type means the workflow re-runs when new commits are pushed to an open PR.
- **`requireScope: false`** — scopes like `feat(auth): ...` are allowed but not required. This keeps the barrier low for contributors.
- The allowed types cover the full Conventional Commits standard. Only `feat` and `fix` trigger version bumps. The rest (`docs`, `style`, `chore`, etc.) are valid but don't create releases.
- After setting up branch protection rules, this check must pass before a PR can be merged. If someone titles their PR "fixed stuff", the bot will block the merge and explain what format is expected.
