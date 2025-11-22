# CLAUDE.md - LUMOS GitHub Action

> **Ecosystem Context:** See [getlumos/lumos/CLAUDE.md](https://github.com/getlumos/lumos/blob/main/CLAUDE.md) for LUMOS ecosystem overview, cross-repo standards, and shared guidelines.

---

## Purpose

GitHub Action for automated LUMOS schema validation and code generation in CI/CD pipelines.

---

## Repository Structure

```
lumos-action/
├── action.yml           # Composite action definition
├── README.md            # Usage documentation
├── CHANGELOG.md         # Version history
├── LICENSE-MIT          # MIT License
└── LICENSE-APACHE       # Apache 2.0 License
```

---

## Key Files

**action.yml** - Composite action with 6 inputs, 4 outputs
- Installs Rust toolchain + LUMOS CLI
- Validates schemas
- Generates code (Rust + TypeScript)
- Detects drift between generated and committed files
- Posts PR comments with diff summaries

**README.md** - Comprehensive usage guide with examples

---

## Marketplace Information

**Name:** LUMOS Generate
**Categories:** Continuous Integration, Code Quality
**Icon:** code (purple)
**URL:** https://github.com/marketplace/actions/lumos-generate

---

## Development

### Making Changes

1. Edit `action.yml` or `README.md`
2. Test changes in a test repository
3. Update `CHANGELOG.md`
4. Create new release with version tag

### Publishing Updates

```bash
# Tag new version
git tag -a v1.0.1 -m "Release v1.0.1 - Bug fixes"
git push origin v1.0.1

# Update major version tag
git tag -fa v1 -m "Latest v1.x release"
git push origin v1 --force

# Create GitHub Release
gh release create v1.0.1 --title "v1.0.1" --notes "..."
# Check "Publish to Marketplace" in release UI
```

---

## Version Strategy

- **v1.0.x** - Patch releases (bug fixes, documentation)
- **v1.x.0** - Minor releases (new features, backward compatible)
- **v2.0.0** - Major releases (breaking changes)

**Users reference:** `@v1` (auto-updates to latest v1.x)

---

## Testing

### Local Testing with `act`

```bash
# Install act (GitHub Actions local runner)
brew install act

# Test the action locally
act -l                    # List workflows
act pull_request          # Test PR workflow
```

### Test in Real Repository

Create a test workflow in any repo:

```yaml
name: Test LUMOS Action
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: getlumos/lumos-action@main  # Test from main branch
        with:
          schema: 'test/schema.lumos'
```

---

## Common Tasks

**Update action logic:**
- Edit `action.yml` steps

**Update documentation:**
- Edit `README.md` examples and usage

**Add new input parameter:**
1. Add to `action.yml` inputs section
2. Document in `README.md` inputs table
3. Use in action steps: `${{ inputs.new-param }}`

**Add new output:**
1. Add to `action.yml` outputs section
2. Set in step: `echo "output-name=value" >> $GITHUB_OUTPUT`
3. Document in `README.md` outputs table

---

## Gotchas

- **Composite actions don't support `pre`/`post` steps** - All logic must be in main steps
- **Use `shell: bash` explicitly** - Required for composite actions
- **Test with different LUMOS versions** - Ensure backward compatibility
- **GitHub Actions cache** - Users may not see updates immediately (cache can be 5-10 min)

---

## Integration with Main Repo

This action uses the LUMOS CLI from crates.io. When the main `getlumos/lumos` repo releases a new CLI version:

1. **No action update needed** - Users specify `version: 'latest'` by default
2. **Pin specific version** - Users can use `version: '0.1.1'` for reproducibility
3. **Test compatibility** - Ensure new CLI versions work with action

---

**Status:** v1.0.0 published to GitHub Marketplace
**Last Updated:** 2025-11-22
**Maintainer:** LUMOS Core Team
