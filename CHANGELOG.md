# Changelog

All notable changes to the LUMOS GitHub Action will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-11-22

### Added

- Initial release of LUMOS GitHub Action
- Auto-install LUMOS CLI (any version from crates.io)
- Validate LUMOS schemas in CI/CD pipelines
- Generate Rust and TypeScript code from schemas
- Detect drift between generated and committed files
- Post PR comments with generation results and diffs
- Configurable failure modes (`fail-on-drift`, `check-only`)
- Support for glob patterns in schema paths
- Working directory customization
- Version pinning for reproducible builds

### Features

- **Inputs:**
  - `schema` - Path to schema files (supports globs)
  - `check-only` - Validation-only mode
  - `version` - LUMOS CLI version selection
  - `working-directory` - Custom working directory
  - `fail-on-drift` - Configurable drift handling
  - `comment-on-pr` - PR comment integration

- **Outputs:**
  - `schemas-validated` - Count of validated schemas
  - `schemas-generated` - Count of generated schemas
  - `drift-detected` - Boolean drift status
  - `diff-summary` - Detailed diff information

### Documentation

- Comprehensive README with usage examples
- Troubleshooting guide
- Workflow tips and patterns
- Monorepo support examples

[1.0.0]: https://github.com/getlumos/lumos-action/releases/tag/v1.0.0
