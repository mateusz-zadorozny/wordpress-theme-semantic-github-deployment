# build-release.sh Template

This script is called by semantic-release via `@semantic-release/exec` during the `prepare` step. It:
1. Updates the `Version:` line in `style.css`
2. Creates a clean ZIP file excluding development files
3. For beta versions, uses a `-beta` directory suffix so WordPress treats it as a separate theme

## Template

```bash
#!/bin/bash
set -e

VERSION="$1"
if [ -z "$VERSION" ]; then
  echo "Usage: bash build-release.sh <version>"
  exit 1
fi

echo "Building release v${VERSION}..."

# Update style.css version
sed -i "s/^Version: .*/Version: ${VERSION}/" style.css

# Beta releases use a separate directory name so WordPress treats them
# as a distinct theme — enables side-by-side testing of stable and beta
if [[ "$VERSION" == *"beta"* ]]; then
  DIR_NAME="{{THEME_SLUG}}-beta"
else
  DIR_NAME="{{THEME_SLUG}}"
fi

# Create clean ZIP with consistent directory name
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
  --exclude='CONTRIBUTING.md' \
  --exclude='.env' \
  --exclude='.env.example' \
  --exclude='tests' \
  --exclude='test-results' \
  --exclude='playwright-report' \
  --exclude='playwright.config.*' \
  . "${THEME_DIR}/"

cd "${TMPDIR}"
zip -qr "${OLDPWD}/{{THEME_SLUG}}.zip" "${DIR_NAME}/"
cd "${OLDPWD}"
rm -rf "${TMPDIR}"

echo "Created {{THEME_SLUG}}.zip (v${VERSION}, dir: ${DIR_NAME})"
```

## Notes

- The `rsync --exclude` list covers common development files. Add project-specific exclusions as needed (e.g., `composer.json`, `docs/`, benchmark files).
- The `sed -i` command uses GNU syntax (no `''` after `-i`) — correct for GitHub Actions (Ubuntu).
- If pre-release is NOT enabled, the beta directory logic still works correctly (the condition simply never matches), so you can include it regardless.
- The ZIP file name is always `{{THEME_SLUG}}.zip` — this must match the `assets.path` in `.releaserc.json`.
- Make this file executable after creating it: `chmod +x build-release.sh`
