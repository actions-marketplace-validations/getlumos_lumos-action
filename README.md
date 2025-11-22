# LUMOS Generate Action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-LUMOS%20Generate-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=github)](https://github.com/marketplace/actions/lumos-generate)

GitHub Action for automatic LUMOS schema generation and validation in CI/CD pipelines.

## Quick Start

```yaml
- uses: actions/checkout@v4
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/**/*.lumos'
```

## Features

- ✅ Auto-install LUMOS CLI (any version)
- ✅ Validate LUMOS schemas
- ✅ Generate Rust + TypeScript code
- ✅ Detect drift between generated and committed files
- ✅ Post PR comments with diff summaries
- ✅ Configurable failure modes

## Usage

### Basic Generation

```yaml
name: LUMOS Generate

on: [push, pull_request]

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate from LUMOS schemas
        uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
```

### Validation Only (PR Checks)

```yaml
name: LUMOS Validate

on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate LUMOS schemas
        uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          check-only: true
          fail-on-drift: true
```

### Advanced Configuration

```yaml
- name: Generate with custom version
  uses: getlumos/lumos-action@v1
  with:
    schema: 'programs/**/schema.lumos'
    version: '0.1.1'
    working-directory: './backend'
    fail-on-drift: false
    comment-on-pr: true
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `schema` | Path to schema files (supports globs) | Yes | - |
| `check-only` | Only validate, do not generate | No | `false` |
| `version` | LUMOS CLI version to install | No | `latest` |
| `working-directory` | Working directory for commands | No | `.` |
| `fail-on-drift` | Fail if drift detected | No | `true` |
| `comment-on-pr` | Post PR comment with results | No | `true` |

## Outputs

| Output | Description |
|--------|-------------|
| `schemas-validated` | Number of schemas validated |
| `schemas-generated` | Number of schemas generated |
| `drift-detected` | Whether drift was detected |
| `diff-summary` | Summary of differences |

## Examples

### Monorepo with Multiple Schemas

```yaml
- name: Generate all schemas
  uses: getlumos/lumos-action@v1
  with:
    schema: |
      programs/nft/schema.lumos
      programs/defi/schema.lumos
      programs/gaming/schema.lumos
```

### Specific Version Pinning

```yaml
- name: Generate with pinned version
  uses: getlumos/lumos-action@v1
  with:
    schema: 'schema.lumos'
    version: '0.1.1'
```

### Custom Failure Behavior

```yaml
- name: Generate with warnings only
  uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/*.lumos'
    fail-on-drift: false  # Only warn, don't fail
```

## Workflow Tips

### Pre-commit Hook Alternative

Use this action as a pre-commit check in CI:

```yaml
on:
  pull_request:
    paths:
      - '**/*.lumos'

jobs:
  check-schemas:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: getlumos/lumos-action@v1
        with:
          schema: '**/*.lumos'
          check-only: true
```

### Auto-commit Generated Files

```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schema.lumos'
    fail-on-drift: false

- name: Commit generated files
  run: |
    git config user.name "LUMOS Bot"
    git config user.email "bot@lumos-lang.org"
    git add .
    git commit -m "chore: Update generated files" || exit 0
    git push
```

## Troubleshooting

### Drift Always Detected

If drift is always detected even when files match:

1. Check line ending settings (CRLF vs LF)
2. Ensure consistent rustfmt version
3. Verify schema paths are correct

### Installation Failures

If LUMOS CLI installation fails:

1. Check version exists on crates.io
2. Verify Rust toolchain is compatible
3. Check network connectivity

## Versioning

This action uses semantic versioning. You can reference it in several ways:

- `getlumos/lumos-action@v1` - Latest v1.x release (recommended)
- `getlumos/lumos-action@v1.0.0` - Specific version
- `getlumos/lumos-action@main` - Latest commit (not recommended for production)

## License

Licensed under either of Apache License, Version 2.0 or MIT license at your option.

## Support

- Documentation: https://lumos-lang.org/tools/github-action
- Issues: https://github.com/getlumos/lumos/issues
- Discussions: https://github.com/getlumos/lumos/discussions
