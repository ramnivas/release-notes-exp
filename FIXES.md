# Fixes to Port to Exograph

This document tracks important fixes and improvements made in this test repository that should be ported back to https://github.com/exograph/exograph.

## Critical Fixes

### 1. Add `contents: write` permission to workflow (if needed)

**File:** `.github/workflows/build-binaries.yml` (Exograph) / `.github/workflows/release.yml` (here)

**Location:** At the workflow level, after the `on:` trigger

**Problem:** Without explicit permissions, the GITHUB_TOKEN may only have read access, causing `gh release create` to fail with:
```
HTTP 403: Resource not accessible by integration
```

**Fix:**
```yaml
permissions:
  contents: write
```

**Why:** GitHub Actions requires `contents: write` permission to create releases. Without this, the workflow fails with a 403 error when trying to call the GitHub API to create releases.

**Note:** Exograph's workflow currently works without explicitly setting permissions because the repository has "Read and write permissions" set as the default in Settings > Actions > General > Workflow permissions. However, adding explicit permissions is a best practice for:
- Clarity and self-documentation
- Portability (works regardless of repository settings)
- Security (explicit is better than implicit)

This fix may not be necessary for Exograph, but is good practice and required for repositories with restrictive default permissions.

---

### 2. Add `fetch-depth: 0` to create-release job

**File:** `.github/workflows/build-binaries.yml` (Exograph) / `.github/workflows/release.yml` (here)

**Location:** In the `create-release` job, the checkout step

**Problem:** Without fetching all tags, the workflow cannot find the previous semantic version tag, causing release notes generation to fail or only show notes from the current commit.

**Fix:**
```yaml
- name: Checkout repository
  uses: actions/checkout@v5
  with:
    fetch-depth: 0  # Fetch all history and tags
```

**Why:** By default, GitHub Actions only fetches 1 commit depth. The release script needs to:
1. List all tags with `git tag --list --sort=-version:refname`
2. Find the previous semantic version tag to use with `--notes-start-tag`

Without full history, only the current tag is available, breaking the comparison logic.

---

## Critical Fixes (Continued)

### 3. Handle edge case: tags point to same commit

**File:** `.github/workflows/build-binaries.yml` (Exograph) / `.github/workflows/release.yml` (here)

**Location:** In the `create-release` job, after finding PREVIOUS_TAG

**Problem:** When multiple tags point to the same commit (or there are no commits between tags), `gh release create --generate-notes --notes-start-tag` fails because there are no commits to generate notes from.

**Fix:**
```bash
# Check if there are any commits between previous and current tag
if [ -n "$PREVIOUS_TAG" ]; then
  COMMIT_COUNT=$(git rev-list --count "$PREVIOUS_TAG".."${{ github.ref_name }}" 2>/dev/null || echo "0")
  echo "Commits between $PREVIOUS_TAG and ${{ github.ref_name }}: $COMMIT_COUNT"

  if [ "$COMMIT_COUNT" = "0" ]; then
    echo "No commits between tags, generating notes from all history"
    PREVIOUS_TAG=""
  fi
fi
```

**Why:** GitHub's `--generate-notes` with `--notes-start-tag` expects there to be commits between the two tags. When there are none, it fails with exit code 1. This can happen during testing or if tags are created on the same commit.

---

## Improvements (Optional)

### 4. Add debug output for troubleshooting

**File:** `.github/workflows/build-binaries.yml` (Exograph) / `.github/workflows/release.yml` (here)

**Location:** In the `create-release` job, in the release notes generation script

**Fix:**
```bash
echo "Current tag: ${{ github.ref_name }}"
echo "All tags:"
git tag --list --sort=-version:refname

# ... after finding PREVIOUS_TAG ...

if [ -n "$PREVIOUS_TAG" ]; then
  echo "Previous tag found: $PREVIOUS_TAG"
  # ... rest of script
else
  echo "No previous tag found, generating notes from all history"
  # ... rest of script
fi
```

**Why:** Makes debugging release note generation issues much easier by showing:
- What tag triggered the workflow
- All available tags
- Which previous tag (if any) was selected

---

## Testing Notes

- Test with multiple tags to ensure release notes are generated correctly between versions
- Test with a single tag to ensure it doesn't fail
- Verify that release notes are scoped to commits between the previous and current tag

---

## Status

- [x] Fix #1 (contents: write permission) applied to test repo
- [x] Fix #2 (fetch-depth: 0) applied to test repo
- [x] Fix #3 (tags on same commit) applied to test repo
- [x] Improvement #4 (debug output) applied to test repo
- [ ] Fix #1 ported to Exograph
- [ ] Fix #2 ported to Exograph
- [ ] Fix #3 ported to Exograph
- [ ] Improvements reviewed for Exograph
