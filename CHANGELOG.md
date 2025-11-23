# Changelog

All notable changes to the LUMOS GitHub Action will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **Documentation:** `docs/multi-language.md` - Comprehensive guide to dual-language generation
  - Why both Rust and TypeScript are generated
  - 3 language separation strategies (move, copy, symlinks)
  - Monorepo patterns (centralized vs distributed)
  - FAQ and feature request tracking
- **Documentation:** `docs/custom-output-paths.md` - Custom output location workarounds
  - 4 core workarounds (move, copy, rename, symlinks)
  - 3 monorepo examples (centralized, package-specific, build artifacts)
  - Advanced dynamic configuration
  - Import path updating strategies
- **Documentation:** "Advanced Usage" section in README
  - Multi-language generation overview
  - Custom output paths summary
  - Monorepo support with matrix strategy
  - Links to all comprehensive guides
- **Documentation:** "Customizing PR Comments" section with 5 inline examples
- **Documentation:** "Understanding Failures" section with decision matrix and error type guide
- **Documentation:** Enhanced "Troubleshooting" section with 6 common scenarios
- **Examples:** `examples/workflows/monorepo-multi-package.yml` - 7 monorepo strategies
  - Matrix generation (parallel, isolated failures)
  - Sequential generation (all packages in one job)
  - Centralized collection (all code in one directory)
  - Smart generation (only changed packages)
  - Auto-commit on main branch
  - Per-package custom naming
  - Validation matrix
- **Examples:** `examples/workflows/separate-rust-typescript.yml` - 9 language separation patterns
  - Simple move to separate directories
  - Rename with schema names
  - Copy instead of move
  - Symbolic links
  - Hierarchical organization
  - Language-specific processing
  - Rust-only and TypeScript-only workflows
  - .gitignore integration
- **Examples:** `examples/workflows/custom-output-organization.yml` - 10 organization patterns
  - Standard project layout (src/generated/)
  - Separate source directories (programs/ vs client/)
  - Build artifacts directory
  - Module-based organization
  - Environment-specific outputs
  - Versioned outputs
  - Dynamic configuration from JSON
  - Workspace-based (Cargo/npm)
  - CI/CD pipeline optimized
  - Git submodule ready
- **Examples:** `examples/workflows/custom-pr-comments.yml` - 5 PR comment customization examples
- **Examples:** `examples/workflows/error-handling.yml` - 6 error handling strategies
- **Examples:** `examples/workflows/diff-parsing.yml` - 6 diff parsing techniques
- Output reference table documenting all output types and formats
- Common error messages documentation
- `diff-summary` output format specification

### Improved

- README now includes Advanced Usage section (multi-language, custom paths, monorepo)
- Troubleshooting guide now includes solutions for 6 common issues
- Clear distinction between validation errors (always fail) vs drift warnings (configurable)
- Documentation of output parsing techniques with regex examples

### Context7 Benchmark Impact

- Q4 (Multi-language Generation): 28 → 80 (+52 points)
- Q5 (PR Comment Customization): 72 → 85 (+13 points)
- Q7 (Custom Output Paths): 65 → 80 (+15 points)
- Q8 (Error Type Documentation): 72 → 85 (+13 points)
- Overall Score: 68.6 → 75.3 (+6.7 points)

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
