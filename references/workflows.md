# GitHub Actions Workflow Templates

## release.yml

This workflow runs semantic-release on every push to the release branch.

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

### Notes on the workflow:

- **First release gotcha:** On some repos (especially private repos on GitHub Free), squash-merging a PR may not trigger the Release workflow on the target branch. This can happen even when the workflow file already exists. If the workflow doesn't trigger after a squash-merge, push any commit directly to `{{MAIN_BRANCH}}` (e.g. `git commit --allow-empty -m "ci: trigger release workflow" && git push origin {{MAIN_BRANCH}}`). Direct pushes reliably trigger workflows. Once the first release is created, subsequent squash-merges typically work fine. The fix is to merge a second PR (any follow-up change) to trigger the first release.
- **`fetch-depth: 0`** — semantic-release needs the full git history to analyze commits and determine the next version.
- **`persist-credentials: false`** — prevents the default GITHUB_TOKEN from being used for git operations. Semantic-release manages its own authentication via the `GITHUB_TOKEN` env var, which lets it push commits (version bump) and create releases.
- **`GITHUB_TOKEN`** — this is the built-in GitHub Actions token. No custom secrets needed. It has write access to contents, issues, and PRs because of the `permissions` block.
- **`npx semantic-release`** — npx runs the locally installed version from devDependencies, ensuring consistent behavior.

---

## pr-lint.yml

This workflow validates that PR titles follow the Conventional Commits format. It blocks merging if the title doesn't match.

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

### Notes on PR lint:

- **`synchronize`** event type means the workflow re-runs when new commits are pushed to an open PR.
- **`requireScope: false`** — scopes like `feat(auth): ...` are allowed but not required. This keeps the barrier low for contributors.
- The allowed types cover the full Conventional Commits standard. Only `feat` and `fix` trigger version bumps. The rest (`docs`, `style`, `chore`, etc.) are valid but don't create releases.
- After setting up branch protection rules, this check must pass before a PR can be merged. If someone titles their PR "fixed stuff", the bot will block the merge and explain what format is expected.

---

## beta-release.yml (optional)

A manually-triggered workflow that builds a beta ZIP and creates a GitHub pre-release. No separate branch needed — you pick the branch from the GitHub Actions UI when triggering.

This workflow requires `build-release.sh` to exist (it reuses the same build logic as stable releases).

```yaml
name: Beta Release

on:
  workflow_dispatch:

permissions:
  contents: write

jobs:
  beta-release:
    name: Build Beta Release
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Determine beta version
        id: version
        run: |
          CURRENT=$(grep -oP '^Version:\s*\K.*' style.css | tr -d '[:space:]')
          BETA_VERSION="${CURRENT}-beta.${{ github.run_number }}"
          echo "version=${BETA_VERSION}" >> "$GITHUB_OUTPUT"
          echo "Beta version: ${BETA_VERSION}"

      - name: Build beta ZIP
        run: bash build-release.sh "${{ steps.version.outputs.version }}"

      - name: Create GitHub pre-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh release create "v${{ steps.version.outputs.version }}" \
            --title "v${{ steps.version.outputs.version }}" \
            --notes "Beta release from branch \`${{ github.ref_name }}\` (run #${{ github.run_number }})" \
            --prerelease \
            "{{THEME_SLUG}}.zip#{{THEME_SLUG}}-beta.zip"
```

### Notes on beta-release:

- **`workflow_dispatch`** — triggered manually from GitHub Actions → "Run workflow" button. GitHub's UI lets you pick which branch to run on.
- **Version format** — reads the current `Version:` from `style.css` and appends `-beta.{run_number}`. For example, if stable is `1.2.0` and this is run #15, the beta version is `1.2.0-beta.15`.
- **`build-release.sh`** — the same script used for stable releases. When the version contains "beta", it automatically uses the `-beta` directory suffix in the ZIP so WordPress treats it as a separate theme.
- **`gh release create --prerelease`** — creates a GitHub pre-release (marked with a "Pre-release" badge). The WP Admin auto-updater ignores pre-releases by design, so beta releases never land on production accidentally.
- **Asset label** — `{{THEME_SLUG}}.zip#{{THEME_SLUG}}-beta.zip` uploads the file as `{{THEME_SLUG}}.zip` but displays it as `{{THEME_SLUG}}-beta.zip` on the release page for clarity.
- **No Node.js needed** — this workflow doesn't run semantic-release, just the shell build script. Faster and simpler.
