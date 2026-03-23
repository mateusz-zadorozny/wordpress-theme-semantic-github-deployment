# beta.sh Template

A local script for quickly building beta ZIP packages. Runs on macOS (developer machine), not in CI. Creates a clean ZIP with `-beta` directory suffix so WordPress treats it as a separate theme — enables side-by-side testing of stable and beta on the same WP instance.

## Template

```bash
#!/bin/bash
set -e

SLUG="{{THEME_SLUG}}"
DIR_NAME="${SLUG}-beta"
OUTPUT="${SLUG}-beta.zip"

echo "Building local beta package..."

TMPDIR=$(mktemp -d)
THEME_DIR="${TMPDIR}/${DIR_NAME}"
mkdir -p "${THEME_DIR}"

rsync -a \
  --exclude='.git' \
  --exclude='.github' \
  --exclude='.claude' \
  --exclude='node_modules' \
  --exclude='.releaserc.json' \
  --exclude='.gitignore' \
  --exclude='package.json' \
  --exclude='package-lock.json' \
  --exclude='build-release.sh' \
  --exclude='beta.sh' \
  --exclude='CONTRIBUTING.md' \
  --exclude='.env' \
  --exclude='.env.example' \
  --exclude='tests' \
  --exclude='test-results' \
  --exclude='playwright-report' \
  --exclude='playwright.config.*' \
  . "${THEME_DIR}/"

# Mark as beta in the ZIP copy (not in working tree)
sed -i '' 's/^Theme Name: .*/& (Beta)/' "${THEME_DIR}/style.css"

cd "${TMPDIR}"
zip -qr "${OLDPWD}/${OUTPUT}" "${DIR_NAME}/"
cd "${OLDPWD}"
rm -rf "${TMPDIR}"

echo "Created ${OUTPUT}"
```

## Notes

- **macOS-compatible** — uses `sed -i ''` (BSD sed). This script runs locally, not in CI.
- **Theme Name suffix** — appends " (Beta)" to the Theme Name in style.css inside the ZIP only. The working tree is untouched. This makes the beta clearly distinguishable in WP Admin → Appearance → Themes.
- **Directory name** — uses `{{THEME_SLUG}}-beta` as the directory inside the ZIP. WordPress uses directory names as theme identifiers, so this is treated as a separate theme from the stable version.
- **No version manipulation** — the beta ZIP uses whatever version is in style.css. For local testing, the exact version doesn't matter — the `-beta` directory name and "(Beta)" in the theme name are enough to distinguish it.
- **rsync exclusions** — same list as `build-release.sh`. Add project-specific exclusions as needed.
- Make executable after creating: `chmod +x beta.sh`
