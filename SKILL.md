---
name: wordpress-theme-semantic-github-deployment
description: Use this skill any time a user wants to automate releases, versioning, or changelogs for a WordPress theme hosted on GitHub. This includes requests to set up semantic-release, add GitHub Actions, enforce conventional commits, auto-bump style.css version numbers, generate changelogs, create a release pipeline, or go from manual/no versioning to automated releases. Applies to all WordPress themes — custom, child, starter. Supports optional beta testing (local beta.sh + GitHub Actions beta trigger), ZIP build artifacts, and WP Admin update widgets. Do NOT use for SSH/FTP deployment, wordpress.org SVN publishing, semantic-release for non-WordPress projects (Node.js/npm), fixing an already-working semantic-release config, or Dependabot setup.
---

# WordPress Theme: Semantic Release & GitHub Deployment

Set up a complete automated release pipeline for any WordPress theme. The developer gets production-grade release automation with zero manual version management — semantic-release handles version bumps, changelogs, and GitHub releases automatically based on PR titles.

## What gets created

| File / Directory | Purpose | When |
|---|---|---|
| `.releaserc.json` | Semantic release configuration | Always |
| `.github/workflows/release.yml` | Auto-release on merge to main | Always |
| `.github/workflows/pr-lint.yml` | Enforce Conventional Commits on PR titles | Always |
| `package.json` | Node.js devDependencies | Always (create or update) |
| `.gitignore` additions | Ignore node_modules etc. | Always |
| `CONTRIBUTING.md` | Developer guide: workflow, PR naming, examples | Always |
| `build-release.sh` | Create clean installable ZIP for each release | If ZIP assets or GitHub beta releases enabled |
| `beta.sh` | Build beta ZIP locally for manual testing | If beta testing enabled |
| `.github/workflows/beta-release.yml` | Manual trigger for GitHub beta pre-releases | If GitHub beta releases enabled |
| `includes/github-updater/*.php` | WP Admin dashboard widget + auto-update | If updater enabled |

## Phase 1: Auto-detection

Before asking questions, silently gather context:

1. **`style.css`** — Read theme header. Extract: Theme Name, Version, Text Domain, Template (parent theme name if child theme)
2. **Git remote** — `git remote get-url origin` → parse GitHub owner and repo name
3. **Git branch** — `git branch --show-current` and check if `main` or `master` exists
4. **`package.json`** — Does one exist? Note existing dependencies
5. **`.releaserc*`** — If semantic-release is already configured, warn and ask before overwriting
6. **`.gitignore`** — Check current ignore rules
7. **Directory structure** — Does `includes/` exist? Does `functions.php` auto-load from includes?

Use detected values as smart defaults in the interview.

## Phase 2: User interview

Present what you detected and ask the user to confirm or adjust. Keep it conversational — don't dump a huge form.

### Core settings (confirm detected defaults):
1. **GitHub repository** — `owner/repo` (from git remote)
2. **Main release branch** — `main` or `master` (from git)

### Optional features (present as choices):

3. **Beta testing** — Local script (`beta.sh`) to quickly build beta ZIPs for manual testing
   - Creates a ZIP with `-beta` directory suffix so WordPress treats it as a separate theme
   - Runs locally (macOS-compatible), no CI needed
   - Optionally also add a **GitHub Actions beta trigger** (`beta-release.yml`) — a "Run workflow" button that creates a proper pre-release on GitHub with a ZIP attached. No merge required — pick any branch from the UI.

4. **ZIP asset on GitHub Releases** — Attach a clean .zip to each release, ready to install in WordPress
   - Excludes dev files (.git, node_modules, config files)

5. **WP Admin auto-updater** — Dashboard widget + integration with WordPress core update system
   - Shows current vs. latest version on the Dashboard
   - One-click "Update now" button
   - If yes, ask: **public or private** repository?
   - Private repos need a GitHub token — the module creates a settings page and admin notice to guide the site admin through configuration

### Confirm before generating:

Show a summary of what will be created and ask the user to confirm:
```
Theme: [Theme Name]
Repo:  [owner/repo]
Main:  [branch]

Features:
  [x] Semantic release + changelog
  [x] PR lint (Conventional Commits)
  [ ] Beta testing (beta.sh)
  [ ] GitHub beta releases (workflow_dispatch)
  [ ] ZIP asset on releases
  [ ] WP Admin updater (private repo)

Files to create: [list]
```

## Phase 3: File generation

Generate files in this order. Read the appropriate reference file before each step.

### Placeholder values

Every template uses these placeholders — resolve them from detection + interview:

| Placeholder | Example | Source |
|---|---|---|
| `{{THEME_SLUG}}` | `my-theme` | Directory name |
| `{{THEME_NAME}}` | `My Theme` | style.css Theme Name |
| `{{REPO_OWNER}}` | `johndoe` | Git remote |
| `{{REPO_NAME}}` | `my-theme` | Git remote |
| `{{MAIN_BRANCH}}` | `main` | Git / user choice |
| `{{TEXT_DOMAIN}}` | `my-theme` | style.css or slug |
| `{{PREFIX}}` | `my_theme` | Snake_case of slug |
| `{{PREFIX_UPPER}}` | `MY_THEME` | Uppercase of prefix |

### Step 1: `.releaserc.json`
Read `references/releaserc.md`. Always uses a single release branch. Choose the appropriate plugins template based on:
- ZIP assets? → uses `build-release.sh` in exec plugin + github assets config
- No ZIP? → uses inline `sed` command in exec plugin

### Step 2: GitHub Actions workflows
Read `references/workflows.md`.
- `release.yml` — triggers on main branch push
- `pr-lint.yml` — validates PR titles against main
- `beta-release.yml` — (if GitHub beta releases enabled) manual workflow_dispatch trigger

### Step 3: `package.json`
If none exists, create a minimal one. If one exists, merge devDependencies.

Required devDependencies:
```json
{
  "semantic-release": "^25.0.0",
  "@semantic-release/changelog": "^6.0.3",
  "@semantic-release/git": "^10.0.1",
  "@semantic-release/exec": "^7.1.0"
}
```

### Step 4: `.gitignore`
Append if not already present:
```
node_modules/
```

### Step 5: `CONTRIBUTING.md`
Read `references/contributing.md`. Customize with theme name, branch names, and enabled features.

### Step 6: `build-release.sh` (if ZIP assets or GitHub beta releases)
Read `references/build-script.md`. Make executable with `chmod +x`. This script is shared between stable releases (via semantic-release) and beta releases (via `beta-release.yml`).

### Step 7: `beta.sh` (if beta testing)
Read `references/beta-script.md`. Make executable with `chmod +x`.

### Step 8: `beta-release.yml` (if GitHub beta releases)
Read `references/workflows.md` — the `beta-release.yml` section. This workflow requires `build-release.sh` to exist. If the user enabled GitHub beta releases but not ZIP assets, still create `build-release.sh` (Step 6).

### Step 9: GitHub updater module (if enabled)
Read `references/github-updater.md`. Create `includes/github-updater/` with PHP files.

For **public repos**: generate loader, client, updater, dashboard-widget (4 files).
For **private repos**: also generate settings and admin-notice (6 files).

After creating the files, ensure `functions.php` loads the module. Check if it already auto-loads from `includes/`. If not, add a `require_once` for the loader.

## Phase 4: Install dependencies

```bash
npm install
```

Verify installation succeeded (check for errors in output).

## Phase 5: Post-setup guide

Present clear, actionable next steps with specific instructions.

### GitHub repository settings:

1. **Squash merge only** — repo Settings → General → Pull Requests:
   - Enable "Allow squash merging" and set default to "Pull request title"
   - Disable "Allow merge commits" and "Allow rebase merging"

2. **Branch protection for `{{MAIN_BRANCH}}`** — Settings → Branches → Add rule:
   - Require pull request before merging
   - Require status checks: "Validate PR title"

### If private repo + auto-updater:

3. **Create a Fine-grained Personal Access Token** at github.com/settings/tokens?type=beta:
   - Name: `{{THEME_NAME}} WP Updater`
   - Repository access: select only the theme repo
   - Permissions: Contents → Read-only
   - Copy the token

4. **Add to wp-config.php** (before "That's all, stop editing!"):
   ```php
   define('{{PREFIX_UPPER}}_GITHUB_TOKEN', 'github_pat_...');
   ```

### Creating the first release:

> **Important: first release gotcha.** On some repos (especially private repos on GitHub Free), squash-merging a PR may not trigger the Release workflow on the target branch. This can happen because the workflow file is being introduced for the first time, or due to GitHub's internal token permissions for PR merges.
>
> **Workaround:** If the workflow doesn't trigger after merging, push any commit directly to `{{MAIN_BRANCH}}`:
> ```bash
> git checkout {{MAIN_BRANCH}} && git pull
> git commit --allow-empty -m "ci: trigger release workflow"
> git push origin {{MAIN_BRANCH}}
> ```
> Direct pushes reliably trigger workflows. Once the first release is created, subsequent squash-merges typically work fine.

5. Commit the setup, push, and create a PR with a conventional title:
   ```
   feat: add automated release pipeline
   ```
6. After squash-merging, check the Actions tab. If the Release workflow didn't trigger, use the direct push workaround above.
7. The first stable version is created automatically.

### Daily workflow cheat sheet:

```
feature-branch → PR (conventional title) → squash merge → auto release

PR title → version bump:
  feat: ...     → minor  (1.1.0 → 1.2.0)
  fix: ...      → patch  (1.2.0 → 1.2.1)
  feat!: ...    → major  (1.2.1 → 2.0.0)
  docs/chore/.. → no release
```

If beta testing:
```
Local beta:   bash beta.sh → upload ZIP to staging WP
GitHub beta:  Actions → Beta Release → Run workflow → pick branch → pre-release created
```
