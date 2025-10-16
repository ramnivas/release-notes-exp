# Release Notes Experiment

This is a minimal setup to test and iterate on the release notes processing from [Exograph](https://github.com/exograph/exograph).

## What was copied

From the Exograph repository, I copied:

**`.github/workflows/release.yml`** - Simplified version of `build-binaries.yml`

- Creates draft releases with auto-generated notes
- Uses `gh release create --generate-notes`
- Finds the previous semantic version tag to generate notes between versions
- Simplified build steps for testing

## How Release Notes Processing Works

The key release notes processing happens in the `create-release` job in `.github/workflows/release.yml`:

1. **Triggered by tags**: The workflow runs when you push a semantic version tag (e.g., `v1.0.0`)

2. **Finds previous tag**: Uses this command to find the previous semantic version:

   ```bash
   git tag --list --sort=-version:refname | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$' | head -2 | tail -1
   ```

3. **Generates notes**: Uses GitHub CLI to create a draft release with auto-generated notes:

   ```bash
   gh release create "$TAG" \
     --draft \
     --generate-notes \
     --notes-start-tag "$PREVIOUS_TAG" \
     --title "Release $TAG"
   ```

4. **Review and publish**: After the workflow runs, you can:
   - Review the generated release notes
   - Edit them if needed
   - Publish the release using the GitHub UI or `gh` CLI

## Testing the Workflow

**Important:** GitHub's auto-generated release notes work with **Pull Requests**, not direct commits. Use PRs with labels for best results.

### Testing with Pull Requests (recommended)

1. Create a feature branch and make changes:

   ```bash
   git checkout -b feat/new-feature
   # Make your changes
   git commit -m "Add new feature"
   git push origin feat/new-feature
   ```

2. Create a PR via GitHub UI or CLI with a **conventional commit title**:

   ```bash
   gh pr create --title "feat: Add cool feature" --body "Description"
   ```

   **Important:** The PR title must follow conventional commits format (e.g., `feat:`, `fix:`, `docs:`)

3. **Labels are added automatically** by the label-prs workflow based on your PR title
   - `feat: ...` → `feat` label
   - `fix: ...` → `fix` label
   - `docs: ...` → `docs` label
   - etc.

4. Merge the PR

5. Create and push a tag:

   ```bash
   git tag v0.2.0
   git push origin v0.2.0
   ```

6. The workflow will:

   - Create a draft release with categorized notes
   - Upload build artifacts (if build succeeds)

7. View the draft release on GitHub, edit if needed, then publish

### Release Note Categories

Based on PR labels (configured in `.github/release.yml`):

- `feat` → 🎉 Features
- `fix` → 🐛 Bug Fixes
- `docs` → 📚 Documentation
- `security` → 🔒 Security
- `perf` → ⚡ Performance
- `refactor` → 🏗️ Refactoring
- `test` → 🧪 Testing
- `ci` → 👷 CI/CD
- `chore` → 🔧 Maintenance

test change
