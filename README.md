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

To test the release notes processing:

1. Make some commits:
   ```bash
   git commit -m "Add new feature"
   git commit -m "Fix bug in module"
   git commit -m "Update documentation"
   ```

2. Create and push a tag:
   ```bash
   git tag v0.1.0
   git push origin v0.1.0
   ```

3. The workflow will:
   - Create a draft release
   - Generate notes from all commits since the last tag
   - Upload build artifacts (if build succeeds)

4. View the draft release on GitHub and review the generated notes
