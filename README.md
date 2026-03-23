# WordPress Theme: Semantic Release & GitHub Deployment

[![My Services](https://img.shields.io/badge/My_Services-shift64.com-d97757?style=for-the-badge)](https://shift64.com)

A [Claude Code skill](https://code.claude.com/docs/en/skills) that sets up a complete automated release pipeline for any WordPress theme in minutes.

## What it does

Tell Claude to set up releases for your WordPress theme and this skill handles everything:

- **Semantic versioning** — automatic version bumps based on PR titles (`feat:` = minor, `fix:` = patch, `feat!:` = major)
- **Changelog generation** — `CHANGELOG.md` updated on every release
- **GitHub Actions** — release workflow + PR title linter (Conventional Commits)
- **style.css version sync** — the `Version:` header in style.css is always updated automatically
- **CONTRIBUTING.md** — developer guide explaining the workflow

### Optional features

- **Beta testing** — local `beta.sh` script for quick beta ZIPs + optional GitHub Actions manual trigger for creating beta pre-releases from any branch (no merge conflicts)
- **ZIP build assets** — clean installable `.zip` attached to each GitHub Release
- **WP Admin auto-updater** — dashboard widget + one-click updates from GitHub (supports both public and private repos)

## Installation

### Per user (available in all your projects)

```bash
git clone https://github.com/mateusz-zadorozny/wordpress-theme-semantic-github-deployment.git \
  ~/.claude/skills/wordpress-theme-semantic-github-deployment
```

### Per project (available only in this repo, shared with your team)

```bash
git clone https://github.com/mateusz-zadorozny/wordpress-theme-semantic-github-deployment.git \
  .claude/skills/wordpress-theme-semantic-github-deployment
```

Then commit `.claude/skills/` to version control so everyone on the team gets the skill.

Start a new Claude Code conversation to make the skill available.

## Usage

Open Claude Code in your WordPress theme's root directory and say something like:

- *"set up semantic release for my WordPress theme"*
- *"add automated versioning and changelog to my theme"*
- *"I want GitHub Actions to auto-release when I merge PRs"*
- *"configure CI/CD with conventional commits for my WP theme"*

The skill will:

1. **Auto-detect** your theme name, git remote, branch, and existing config
2. **Interview** you about optional features (beta testing, ZIP assets, WP updater)
3. **Generate** all configuration files with your theme's details filled in
4. **Install** npm dependencies
5. **Guide** you through GitHub repo settings (squash merge, branch protection)

## Files generated

| File | Purpose | When |
|---|---|---|
| `.releaserc.json` | Semantic release configuration | Always |
| `.github/workflows/release.yml` | Auto-release on merge | Always |
| `.github/workflows/pr-lint.yml` | PR title validation | Always |
| `package.json` | Node.js devDependencies | Always |
| `CONTRIBUTING.md` | Developer workflow guide | Always |
| `build-release.sh` | ZIP builder for releases | If ZIP or GitHub beta enabled |
| `beta.sh` | Local beta ZIP builder | If beta testing enabled |
| `.github/workflows/beta-release.yml` | Manual beta release trigger | If GitHub beta enabled |
| `includes/github-updater/*.php` | WP dashboard widget + auto-update | If updater enabled |

## Daily workflow after setup

```
feature-branch  -->  PR (conventional title)  -->  squash merge  -->  auto release

PR title examples:
  feat: add WooCommerce support     --> minor bump (1.1.0 -> 1.2.0)
  fix: correct footer alignment     --> patch bump (1.2.0 -> 1.2.1)
  feat!: drop IE11 support          --> major bump (1.2.1 -> 2.0.0)
  docs: update README               --> no release
```

With beta testing enabled:
```
Local:   bash beta.sh  -->  upload ZIP to staging WP
GitHub:  Actions > Beta Release > Run workflow > pick branch  -->  pre-release with ZIP
```

## What gets installed in your project

The skill adds these npm devDependencies to your theme's `package.json`:

| Package | Version | Purpose |
|---|---|---|
| `semantic-release` | ^25.0.0 | Core engine — analyzes commits, calculates versions, orchestrates plugins |
| `@semantic-release/changelog` | ^6.0.3 | Generates and updates `CHANGELOG.md` |
| `@semantic-release/git` | ^10.0.1 | Commits version bumps and changelog back to the repo |
| `@semantic-release/exec` | ^7.1.0 | Runs `build-release.sh` (or `sed`) to update `style.css` version |

These are **devDependencies only** — they run in GitHub Actions CI, never on your WordPress site. Your theme's production code is not affected.

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- A WordPress theme hosted on GitHub
- Node.js (for semantic-release — only needed locally for `npm install`, then CI handles the rest)

## Author

Built by [Mateusz Zadorozny](https://shift64.com) — WordPress & WooCommerce development, B2B e-commerce, CI/CD automation.

## Changelog

### 1.0.1

- **Replaced pre-release branch workflow with local beta testing** — no more dedicated `beta` branch and merge conflicts. Beta is now handled via `beta.sh` (local macOS script) and an optional `workflow_dispatch` GitHub Action that builds beta pre-releases from any branch on demand.
- Added `references/beta-script.md` template for `beta.sh`
- Added `beta-release.yml` workflow template to `references/workflows.md`
- Removed `{{BETA_BRANCH}}` placeholder — single release branch only
- Updated all reference files and CONTRIBUTING.md template to reflect new beta flow

### 1.0.0

- Initial release — semantic-release pipeline, PR linter, ZIP assets, WP Admin auto-updater, pre-release branch support

## License

MIT
