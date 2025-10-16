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

### 4. Use `gh release upload` instead of `svenstaro/upload-release-action`

**File:** `.github/workflows/build-binaries.yml` (Exograph) / `.github/workflows/release.yml` (here)

**Location:** In the `build-artifacts` job (and other build jobs), the "Upload zip to release" step

**Problem:** `svenstaro/upload-release-action` cannot upload to draft releases. It searches for a release by tag, but draft releases aren't tagged yet (they have "untagged-..." URLs). This causes it to create a NEW published release with no notes, resulting in duplicate releases.

**Fix:**
Replace:
```yaml
- name: Upload zip to release
  if: startsWith(github.ref, 'refs/tags/')
  uses: svenstaro/upload-release-action@v2
  with:
    repo_token: ${{ secrets.GITHUB_TOKEN }}
    file: dist/exograph-${{matrix.target}}.zip
    asset_name: exograph-${{matrix.target}}.zip
    tag: ${{ github.ref }}
    make_latest: false
```

With:
```yaml
- name: Upload zip to draft release
  if: startsWith(github.ref, 'refs/tags/')
  run: |
    gh release upload "${{ github.ref_name }}" dist/exograph-${{matrix.target}}.zip --clobber
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Why:** `gh release upload` can upload to draft releases by tag name. The `--clobber` flag allows overwriting existing assets.

**Benefit:** Eliminates duplicate releases and manual cleanup while preserving the ability to review and edit notes before publishing.

---

## Issues Under Investigation

### Double Release Creation & "untagged-..." URLs

**Root Cause:** `gh release create TAG --draft` creates a draft release that is NOT immediately bound to the tag. The draft gets an "untagged-..." URL initially and only gets properly tagged when published.

**Evidence:** Both test repo and Exograph show this behavior:
- Test repo: `https://github.com/ramnivas/release-notes-exp/releases/tag/untagged-c02993dbc489e14e0fea`
- Exograph: `https://github.com/exograph/exograph/releases/tag/untagged-c969982411002041eecc`

**Problem:** `svenstaro/upload-release-action` looks for a release by tag. Since the draft isn't tagged yet, it can't find it and creates a NEW published release with the tag but NO notes.

**Result:** Two releases for the same tag:
1. Draft with notes but "untagged-..." URL (from `create-release` job)
2. Published with correct tag but NO notes (from `svenstaro/upload-release-action`)

**Exograph's Approach:** The draft release is created, assets are uploaded (creating the duplicate), then someone **manually publishes the draft release** through the GitHub UI or `gh` CLI. The comment in their workflow says:
```
# After this workflow is run:
# - Review the release notes and binaries in the draft release
# - Using the Github UI (or the `gh` cli), publish the release
```

**Current Solution (deviation from Exograph):** Added `needs: create-release` to ensure draft exists before upload, but this doesn't solve the dual-release issue - it just ensures timing.

**Solution Implemented:** Use `gh release upload` instead of `svenstaro/upload-release-action`:

```yaml
- name: Upload zip to draft release
  if: startsWith(github.ref, 'refs/tags/')
  run: |
    gh release upload "${{ github.ref_name }}" dist/release-notes-exp.zip --clobber
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Why this works:** `gh release upload` can upload to draft releases by tag name, even though drafts have "untagged-..." URLs. This prevents the duplicate release.

**Workflow:**
1. Tag pushed → draft release created with notes
2. Assets uploaded to the draft
3. Manual review/edit of notes via GitHub UI
4. Click "Publish release" → one release with notes and assets

**For Exograph:** Replace `svenstaro/upload-release-action` with `gh release upload` to eliminate duplicate releases. This removes the manual cleanup step currently required while keeping the draft review capability.

---

## Important: Release Notes Require Pull Requests with Labels

GitHub's `--generate-notes` feature works with **Pull Requests with labels**, not direct commits. To get detailed, categorized release notes:

### Automatic Labeling

The `.github/workflows/label-prs.yml` workflow automatically labels PRs based on conventional commit prefixes in the first commit:
- `feat:` or `feat(scope):` → adds `feat` label
- `fix:` or `fix(scope):` → adds `fix` label
- `docs:` or `docs(scope):` → adds `docs` label
- etc.

### Manual Process

1. **Create feature branch** and make commits
2. **Create PR** with descriptive title
3. **Add labels** to PR (feat, fix, docs, etc.) - these determine which section the PR appears under
4. **Merge PR** to main
5. **Tag and push** - the tag triggers the workflow

The `.github/release.yml` configuration maps PR labels to release note sections:
- `feat` → 🎉 Features
- `fix` → 🐛 Bug Fixes
- `docs` → 📚 Documentation
- etc.

**Without PRs:** You'll only get "Full Changelog" link (what you're seeing now).
**With PRs:** You'll get nicely formatted, categorized notes like Exograph's releases.

This is Exograph's workflow - all changes go through PRs with appropriate labels.

---

## Testing Notes

- Test with multiple tags to ensure release notes are generated correctly between versions
- Test with a single tag to ensure it doesn't fail
- Verify that release notes are scoped to commits between the previous and current tag
- Verify that only one release is created (not duplicates)
- **Use PRs with labels** to test categorized release notes

---

## Improvement: Use Maintained Action for PR Labeling

**Alternative Approach (test repo):** Replace custom labeling code with `bcoe/conventional-release-labels`:

```yaml
- uses: bcoe/conventional-release-labels@v1
  with:
    type_labels: >
      {
        "feat": "feat",
        "fix": "fix",
        "docs": "docs",
        ...
      }
```

**Benefits:**
- Maintained by the community
- Well-tested and widely used
- No custom code to maintain
- Handles edge cases automatically

**Important Difference from Exograph:**
- **Exograph's custom code:** Labels based on **first commit message** in PR
- **bcoe action:** Labels based on **PR title** only
- **Trade-off:** Simpler but requires PR titles to follow conventional commits (e.g., "feat: Add feature")

**For Exograph:** Replace custom labeling code with this maintained action. The team already follows conventional commit format in PR titles (as seen in recent PRs), making this a safe simplification. If commit-based labeling (vs title-based) becomes needed later, can revert.

---

## Improvement: Custom Release Note Format

**Alternative Approach (test repo):** Replace GitHub's native `--generate-notes` with `mikepenz/release-changelog-builder-action`:

**Problem:** GitHub's native release notes include author and PR number in format: `"feat: Add feature by @author in #123"`. For cleaner notes, you may want to omit the author: `"feat: Add feature #123"`.

**Solution:** Use `mikepenz/release-changelog-builder-action` with custom configuration.

### Step 1: Create Configuration File

Create `.github/release-notes-config.json`:

```json
{
  "template": "#{{CHANGELOG}}",
  "pr_template": "- #{{TITLE}} ##{{NUMBER}}",
  "categories": [
    {
      "title": "## 🚨 Breaking Changes",
      "labels": ["breaking", "breaking-change"]
    },
    {
      "title": "## 🎉 Features",
      "labels": ["feat", "feature", "enhancement"]
    },
    {
      "title": "## 🐛 Bug Fixes",
      "labels": ["bug", "fix", "bugfix"]
    },
    {
      "title": "## 🔒 Security",
      "labels": ["security"]
    },
    {
      "title": "## 📚 Documentation",
      "labels": ["documentation", "docs"]
    },
    {
      "title": "## 🎨 Style",
      "labels": ["style"]
    },
    {
      "title": "## ⚡ Performance",
      "labels": ["performance", "perf"]
    },
    {
      "title": "## 🏗️ Refactoring",
      "labels": ["refactor", "refactoring"]
    },
    {
      "title": "## 🧪 Testing",
      "labels": ["test", "testing"]
    },
    {
      "title": "## 🔨 Build System",
      "labels": ["build"]
    },
    {
      "title": "## 👷 CI/CD",
      "labels": ["ci", "cd"]
    },
    {
      "title": "## 🔧 Maintenance",
      "labels": ["chore", "dependencies", "maintenance"]
    },
    {
      "title": "## ⏪ Reverts",
      "labels": ["revert"]
    },
    {
      "title": "## 🚀 Other Changes",
      "labels": []
    }
  ],
  "label_extractor": [
    {
      "pattern": ".*",
      "on_property": "title"
    }
  ],
  "ignore_labels": ["ignore-for-release", "release"],
  "ignore_authors": ["dependabot", "dependabot[bot]"]
}
```

**Key configuration options:**
- `"template": "#{{CHANGELOG}}"` - Removes the default "Uncategorized" section that appears at the bottom
- `"pr_template": "- #{{TITLE}} ##{{NUMBER}}"` - Formats each PR as a single line without author (use `##{{NUMBER}}` to get `#123` format)
- `"labels": []` - Empty array in "Other Changes" category acts as catch-all for PRs without specific labels
- `"label_extractor"` - Extracts labels from PR title (for use with conventional commits)

### Step 2: Update Workflow

In the create-release job, find where `PREVIOUS_TAG` is determined and add output:

```yaml
- name: Find previous semantic version tag
  id: previous_tag
  run: |
    # ... existing logic to find PREVIOUS_TAG ...

    if [ -n "$PREVIOUS_TAG" ]; then
      echo "Previous tag found: $PREVIOUS_TAG"
      echo "tag=$PREVIOUS_TAG" >> $GITHUB_OUTPUT
    else
      echo "No previous tag found or no commits between tags"
      echo "tag=" >> $GITHUB_OUTPUT
    fi
```

Then replace the release creation section:

```yaml
- name: Generate release notes with custom format
  id: changelog
  uses: mikepenz/release-changelog-builder-action@v5
  with:
    configuration: .github/release-notes-config.json
    fromTag: ${{ steps.previous_tag.outputs.tag }}
    toTag: ${{ github.ref_name }}
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

- name: Create draft release with custom notes
  run: |
    gh release create "${{ github.ref_name }}" \
      --draft \
      --notes "${{ steps.changelog.outputs.changelog }}" \
      --title "Release ${{ github.ref_name }}"
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Replace this old code:**
```yaml
gh release create "${{ github.ref_name }}" \
  --draft \
  --generate-notes \
  --notes-start-tag "$PREVIOUS_TAG" \
  --title "Release ${{ github.ref_name }}"
```

### Step 3: Remove Obsolete Files

Delete `.github/release.yml` - it's only used by GitHub's native `--generate-notes`, which we're no longer using.

### Important Notes

**CRITICAL:** The configuration **must** be in a separate JSON file. Inline JSON in the workflow YAML does not work - the action fails to parse it and silently falls back to default templates (which include author and multi-line format). You'll see this warning in logs:
```
⚠️ Configuration provided, but it couldn't be found. Fallback to Defaults.
```

**Format Examples:**
- Current (with author): `- feat: Add feature by @ramnivas in #123`
- New (without author): `- feat: Add feature #123`

**Benefits:**
- Full control over release note formatting
- Can omit author (`#{{AUTHOR}}`) for cleaner notes
- Removes "Uncategorized" section at bottom of notes
- Maintains all categorization with emojis
- Actively maintained action (~800 stars)

**Trade-offs:**
- Replaces GitHub's native `--generate-notes` (which is simpler)
- Category configuration now lives in `.github/release-notes-config.json` instead of `.github/release.yml`
- Adds one more file to maintain
- Need to manually sync categories if using both this repo and another that uses native notes

**For Exograph:** Apply this change for cleaner, more professional release notes without author attribution. While this adds complexity (separate config file vs inline `--generate-notes`), the improved formatting is worth it.

---

## Status

**Applied to test repo:**
- [x] Fix #1 (contents: write permission)
- [x] Fix #2 (fetch-depth: 0)
- [x] Fix #3 (tags on same commit edge case)
- [x] Fix #4 (use gh release upload)
- [x] Improvement (use bcoe/conventional-release-labels)
- [x] Improvement (debug output)
- [x] Improvement (custom release note format)

**To port to Exograph:**
- [ ] Fix #1 - Review if needed (may not be necessary, but good practice for portability)
- [ ] Fix #2 - **CRITICAL** - Add fetch-depth: 0 to enable proper tag detection
- [ ] Fix #3 - Add commit count check for edge case handling
- [ ] Fix #4 - **IMPORTANT** - Replace svenstaro/upload-release-action with gh release upload
- [ ] Improvement - Replace custom PR labeling code with bcoe/conventional-release-labels
- [ ] Improvement - Add debug output for easier troubleshooting
- [ ] Improvement - Apply custom release note format (mikepenz action)
