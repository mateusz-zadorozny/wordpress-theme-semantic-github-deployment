# CONTRIBUTING.md Template

Generate this file with the placeholders replaced. Adjust the "Optional features" sections based on what the user enabled.

---

```markdown
# Release Automation & Versioning (WordPress Theme)

This document describes the development workflow and release process for **{{THEME_NAME}}**. We use **Semantic Release** with **GitHub Actions** to automatically manage version numbers, changelogs, and GitHub releases.

## How It Works (TL;DR)

1. **Never change versions manually** in `style.css` or `package.json` — the bot handles it.
2. Work on feature branches.
3. Name your Pull Requests using the **Conventional Commits** format (e.g., `feat: add slider`, `fix: header spacing`).
4. After merging to `{{MAIN_BRANCH}}`, the automation:
   - Calculates the next version number
   - Updates `style.css` and `package.json`
   - Generates a `CHANGELOG.md` entry
   - Creates a GitHub Release

---

## Daily Workflow

### 1. Create a Feature Branch

Branch off `{{MAIN_BRANCH}}` with a descriptive name:
```bash
git checkout -b feat/new-header
```

Commit messages within your branch are free-form — only the PR title matters for versioning.

### 2. Open a Pull Request

When opening a PR to `{{MAIN_BRANCH}}`, the **title** determines the version bump. We use **Squash & Merge**, so only the PR title ends up in the commit history.

**Title format:** `type(optional-scope): short description`

| Type | Meaning | Version Bump | Example |
|:---|:---|:---|:---|
| `feat` | New feature | Minor (1.1.0 → 1.2.0) | `feat: add WooCommerce support` |
| `fix` | Bug fix | Patch (1.2.0 → 1.2.1) | `fix: correct footer alignment` |
| `perf` | Performance improvement | Patch | `perf: optimize image loading` |
| `docs` | Documentation only | No release | `docs: update README` |
| `style` | Formatting, CSS | No release | `style: reformat header styles` |
| `refactor` | Code restructuring | No release | `refactor: simplify template logic` |
| `chore` | Maintenance | No release | `chore: update dependencies` |
| `test` | Tests | No release | `test: add checkout flow tests` |
| `ci` | CI/CD changes | No release | `ci: update release workflow` |
| `feat!` | Breaking change | Major (1.2.1 → 2.0.0) | `feat!: drop IE11 support` |

> **PR Linter Bot:** If the PR title doesn't follow this format, the bot blocks the merge. Fix the title and the check re-runs automatically.

### 3. Merge

Click **Squash and merge**. GitHub squashes all your commits into one, using the PR title as the commit message on `{{MAIN_BRANCH}}`.

The release workflow runs automatically.

---

## What Happens on Release

When a PR is merged to `{{MAIN_BRANCH}}`:

1. **Commit analysis** — semantic-release reads the squashed commit message to determine the bump type
2. **Version calculation** — bumps major, minor, or patch based on the commit type
3. **File updates:**
   - `style.css` → `Version: X.Y.Z` header updated
   - `package.json` → `version` field updated
   - `CHANGELOG.md` → new entry prepended
4. **Git commit** — changes committed with `chore(release): X.Y.Z [skip ci]`
5. **Git tag** — `vX.Y.Z` tag created
6. **GitHub Release** — release created with auto-generated notes

```
REPLACE_WITH_BETA_SECTION_OR_REMOVE
```

```
REPLACE_WITH_ZIP_SECTION_OR_REMOVE
```

---

## Setup Reference

### Dependencies

```bash
npm install --save-dev semantic-release @semantic-release/changelog @semantic-release/git @semantic-release/exec
```

### Key Files

| File | Purpose |
|---|---|
| `.releaserc.json` | Semantic release configuration |
| `.github/workflows/release.yml` | Release automation |
| `.github/workflows/pr-lint.yml` | PR title validation |
| `package.json` | Node.js dependencies |

### GitHub Repository Settings

- **Merge method:** Squash and merge only (Settings → General → Pull Requests)
- **Branch protection for `{{MAIN_BRANCH}}`:**
  - Require pull request reviews
  - Require status checks: "Validate PR title"
```

---

## Conditional sections

### If pre-release is enabled, replace `REPLACE_WITH_BETA_SECTION_OR_REMOVE` with:

```markdown
## Pre-release (Beta) Channel

For features that need testing before a stable release:

1. Create a PR targeting `{{BETA_BRANCH}}` instead of `{{MAIN_BRANCH}}`
2. After squash-merging, a pre-release is created: `v1.3.0-beta.1`
3. Subsequent merges to `{{BETA_BRANCH}}` increment: `beta.2`, `beta.3`, etc.
4. When ready for stable release: create a PR from `{{BETA_BRANCH}}` → `{{MAIN_BRANCH}}`
5. After merging, the stable version `v1.3.0` is released

```
feature → PR to {{BETA_BRANCH}} → v1.3.0-beta.1
{{BETA_BRANCH}} → PR to {{MAIN_BRANCH}} → v1.3.0 (stable)
```
```

### If pre-release is NOT enabled, remove the placeholder line entirely.

### If ZIP assets are enabled, replace `REPLACE_WITH_ZIP_SECTION_OR_REMOVE` with:

```markdown
## Release Assets

Each GitHub Release includes a `{{THEME_SLUG}}.zip` file — a clean, installable theme package with all development files excluded. You can download and install it directly via WordPress Admin → Themes → Add New → Upload Theme.
```

### If ZIP assets are NOT enabled, remove the placeholder line entirely.
