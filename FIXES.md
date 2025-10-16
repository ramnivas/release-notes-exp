# Fixes to Port to Exograph

This document tracks important fixes and improvements made in this test repository that should be ported back to https://github.com/exograph/exograph.

## Critical Fixes

### 1. Add `fetch-depth: 0` to create-release job

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

## Improvements (Optional)

### 2. Handle edge case: only one tag exists

**File:** `.github/workflows/build-binaries.yml` (Exograph) / `.github/workflows/release.yml` (here)

**Location:** In the `create-release` job, after finding PREVIOUS_TAG

**Problem:** When only one tag exists, the script finds the current tag as the "previous" tag, causing issues.

**Fix:**
```bash
# If PREVIOUS_TAG is the same as current tag, treat as no previous tag
if [ "$PREVIOUS_TAG" = "${{ github.ref_name }}" ]; then
  PREVIOUS_TAG=""
fi
```

**Why:** The command `head -2 | tail -1` returns the first tag when there's only one tag, which is the current tag being released.

---

### 3. Add debug output for troubleshooting

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

- [x] Fix #1 applied to test repo
- [x] Improvement #2 applied to test repo
- [x] Improvement #3 applied to test repo
- [ ] Fix #1 ported to Exograph
- [ ] Improvements reviewed for Exograph
