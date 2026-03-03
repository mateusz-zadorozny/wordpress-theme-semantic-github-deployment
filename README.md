# WordPress Theme: Semantic Release & GitHub Deployment

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that sets up a complete automated release pipeline for any WordPress theme in minutes.

## What it does

Tell Claude to set up releases for your WordPress theme and this skill handles everything:

- **Semantic versioning** — automatic version bumps based on PR titles (`feat:` = minor, `fix:` = patch, `feat!:` = major)
- **Changelog generation** — `CHANGELOG.md` updated on every release
- **GitHub Actions** — release workflow + PR title linter (Conventional Commits)
- **style.css version sync** — the `Version:` header in style.css is always updated automatically
- **CONTRIBUTING.md** — developer guide explaining the workflow

### Optional features

- **Pre-release channel** — beta branch producing versions like `1.3.0-beta.1`
- **ZIP build assets** — clean installable `.zip` attached to each GitHub Release
- **WP Admin auto-updater** — dashboard widget + one-click updates from GitHub (supports both public and private repos)

## Installation

Clone this repo into your Claude Code skills directory:

```bash
git clone https://github.com/mateusz-zadorozny/wordpress-theme-semantic-github-deployment.git \
  ~/.claude/skills/wordpress-theme-semantic-github-deployment
```

That's it. The skill is available immediately in all your Claude Code sessions.

## Usage

Open Claude Code in your WordPress theme's root directory and say something like:

- *"set up semantic release for my WordPress theme"*
- *"add automated versioning and changelog to my theme"*
- *"I want GitHub Actions to auto-release when I merge PRs"*
- *"configure CI/CD with conventional commits for my WP theme"*

The skill will:

1. **Auto-detect** your theme name, git remote, branch, and existing config
2. **Interview** you about optional features (beta channel, ZIP assets, WP updater)
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
| `build-release.sh` | ZIP builder for releases | If ZIP enabled |
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

With pre-release enabled:
```
feature  -->  PR to beta  -->  v1.3.0-beta.1
beta     -->  PR to main  -->  v1.3.0 (stable)
```

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- A WordPress theme hosted on GitHub
- Node.js (for semantic-release)

## License

MIT
