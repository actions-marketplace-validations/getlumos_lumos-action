# Custom Output Locations

## Default Behavior

LUMOS generates files in the **same directory** as the schema file.

**Example:**
```
schemas/
├── player.lumos       # Your schema
├── generated.rs       # Generated here ✓
└── generated.ts       # Generated here ✓
```

**This cannot be changed directly.** The action does not support a custom output path parameter.

## Supported: Working Directory

You can change the **working directory** where the action runs:

```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/*.lumos'
    working-directory: './packages/core'  # Run from this directory
```

This affects the **current working directory** for commands, but generated files still appear next to schema files.

## Not Supported: Custom Output Path Parameter

**Requested feature:**
```yaml
# ❌ This does NOT work
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/*.lumos'
    output: 'generated/'  # Not supported
```

**Track:** getlumos/lumos#XX - Custom output path support

## Workarounds

Since custom output paths aren't supported natively, use post-generation file organization.

### Workaround 1: Post-Generation Move

Move generated files to custom location after generation:

```yaml
name: Custom Output Location

on: [push, pull_request]

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'

      - name: Move to custom location
        run: |
          mkdir -p custom/output/path

          # Move all generated Rust files
          find schemas -name "generated.rs" \
            -exec mv {} custom/output/path/ \;

          # Move all generated TypeScript files
          find schemas -name "generated.ts" \
            -exec mv {} custom/output/path/ \;

      - name: Update imports (if needed)
        run: |
          # Example: Update Rust imports
          sed -i 's|schemas/generated|custom/output/path/generated|g' \
            src/**/*.rs
```

### Workaround 2: Copy Instead of Move

Keep originals in place, copy to custom location:

```yaml
- name: Copy to custom location
  run: |
    mkdir -p build/generated

    # Copy all generated files
    find schemas -name "generated.*" \
      -exec cp {} build/generated/ \;
```

**Use case:** When you need generated files in both locations (e.g., local development + build artifacts).

### Workaround 3: Rename with Schema Name

Prevent filename conflicts by prefixing with schema name:

```yaml
- name: Rename and move with schema names
  run: |
    mkdir -p output/rust output/typescript

    find schemas -name "*.lumos" | while read schema; do
      schema_name=$(basename "$schema" .lumos)
      schema_dir=$(dirname "$schema")

      # Rename and move Rust file
      if [ -f "$schema_dir/generated.rs" ]; then
        mv "$schema_dir/generated.rs" \
           "output/rust/${schema_name}.rs"
      fi

      # Rename and move TypeScript file
      if [ -f "$schema_dir/generated.ts" ]; then
        mv "$schema_dir/generated.ts" \
           "output/typescript/${schema_name}.ts"
      fi
    done
```

**Result:**
```
output/
├── rust/
│   ├── player.rs
│   ├── inventory.rs
│   └── game_state.rs
└── typescript/
    ├── player.ts
    ├── inventory.ts
    └── game_state.ts
```

### Workaround 4: Symbolic Links

Create symlinks in desired location:

```yaml
- name: Create symlinks to generated code
  run: |
    mkdir -p src/generated

    # Create symlinks to all generated files
    find schemas -name "generated.*" | while read file; do
      ln -sf "$(realpath $file)" src/generated/
    done
```

**Benefits:**
- Files stay in original location (next to schemas)
- Accessible from custom path via symlinks
- No duplicate files

**Drawbacks:**
- Symlinks may not work on Windows
- Git handling of symlinks varies

## Monorepo Examples

### Example 1: Centralized Generated Code

Collect all generated code in one central directory:

```yaml
name: Centralized Output

on: [push, pull_request]

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/*/schemas/*.lumos'

      - name: Centralize all generated code
        run: |
          mkdir -p generated/{rust,typescript}

          # Copy all Rust files with package names
          find packages -name "generated.rs" | while read file; do
            package=$(echo "$file" | cut -d'/' -f2)
            cp "$file" "generated/rust/${package}.rs"
          done

          # Copy all TypeScript files with package names
          find packages -name "generated.ts" | while read file; do
            package=$(echo "$file" | cut -d'/' -f2)
            cp "$file" "generated/typescript/${package}.ts"
          done

      - name: Commit centralized code
        run: |
          git add generated/
          git commit -m "chore: Update centralized generated code" || exit 0
```

**Directory structure:**
```
generated/
├── rust/
│   ├── auth.rs
│   ├── payments.rs
│   └── users.rs
└── typescript/
    ├── auth.ts
    ├── payments.ts
    └── users.ts
```

### Example 2: Package-Specific Output Directories

Each package has its own generated code directory:

```yaml
name: Package-Specific Outputs

on: [push, pull_request]

jobs:
  generate:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package: [auth, payments, users]
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'

      - name: Organize ${{ matrix.package }} outputs
        run: |
          cd packages/${{ matrix.package }}

          # Create output directory
          mkdir -p generated

          # Move files to output directory
          mv schemas/generated.rs generated/ 2>/dev/null || true
          mv schemas/generated.ts generated/ 2>/dev/null || true
```

**Directory structure:**
```
packages/
├── auth/
│   ├── schemas/user.lumos
│   └── generated/
│       ├── generated.rs
│       └── generated.ts
├── payments/
│   ├── schemas/transaction.lumos
│   └── generated/
│       ├── generated.rs
│       └── generated.ts
```

### Example 3: Build Artifacts Directory

Copy generated code to build directory for deployment:

```yaml
- name: Prepare build artifacts
  run: |
    mkdir -p build/artifacts/{rust,typescript}

    # Collect all generated Rust code
    find . -name "generated.rs" | while read file; do
      # Extract package/module name from path
      module=$(echo "$file" | sed 's|./packages/||' | sed 's|/schemas/.*||')

      # Copy to build artifacts
      cp "$file" "build/artifacts/rust/${module}.rs"
    done

    # Collect all generated TypeScript code
    find . -name "generated.ts" | while read file; do
      module=$(echo "$file" | sed 's|./packages/||' | sed 's|/schemas/.*||')
      cp "$file" "build/artifacts/typescript/${module}.ts"
    done

- name: Upload artifacts
  uses: actions/upload-artifact@v4
  with:
    name: generated-code
    path: build/artifacts/
```

## Advanced: Dynamic Path Configuration

Use environment variables or config files for flexible output paths:

```yaml
- name: Load output configuration
  id: config
  run: |
    # Read from config file
    RUST_OUTPUT=$(jq -r '.output.rust' .lumos-config.json)
    TS_OUTPUT=$(jq -r '.output.typescript' .lumos-config.json)

    echo "rust_output=$RUST_OUTPUT" >> $GITHUB_OUTPUT
    echo "ts_output=$TS_OUTPUT" >> $GITHUB_OUTPUT

- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/**/*.lumos'

- name: Move to configured paths
  run: |
    mkdir -p "${{ steps.config.outputs.rust_output }}"
    mkdir -p "${{ steps.config.outputs.ts_output }}"

    find schemas -name "generated.rs" \
      -exec mv {} "${{ steps.config.outputs.rust_output }}/" \;

    find schemas -name "generated.ts" \
      -exec mv {} "${{ steps.config.outputs.ts_output }}/" \;
```

**`.lumos-config.json`:**
```json
{
  "output": {
    "rust": "src/generated/rust",
    "typescript": "src/generated/typescript"
  }
}
```

## Update Import Paths

After moving files, you may need to update imports:

```yaml
- name: Update Rust imports
  run: |
    # Update all Rust files to use new path
    find src -name "*.rs" -exec sed -i \
      's|use schemas::generated|use generated::rust|g' {} \;

- name: Update TypeScript imports
  run: |
    # Update all TypeScript files
    find src -name "*.ts" -exec sed -i \
      's|from "../schemas/generated"|from "../generated/typescript/generated"|g' {} \;
```

## Gitignore Patterns

If using custom output directories, update `.gitignore`:

```gitignore
# Generated code in custom locations
generated/
build/artifacts/
src/generated/
packages/*/generated/

# Keep schemas in git
!schemas/
!**/*.lumos
```

## Common Patterns Summary

| Pattern | Use Case | Complexity |
|---------|----------|------------|
| Post-generation move | Simple custom path | Low |
| Copy to build dir | Deploy artifacts | Low |
| Rename with schema name | Multiple schemas | Medium |
| Symbolic links | Dual access | Medium |
| Centralized monorepo | All packages in one place | Medium |
| Package-specific | Per-package organization | High |
| Dynamic config | Flexible configuration | High |

## FAQ

**Q: Can I specify a custom output directory?**
A: Not directly. Use post-generation file organization (move/copy).

**Q: Does `working-directory` change output location?**
A: No, it only changes where commands run. Files still generate next to schemas.

**Q: What if I have multiple schemas?**
A: Each schema generates its own `generated.rs`/`generated.ts` pair in its directory.

**Q: How do I prevent filename conflicts?**
A: Rename files with schema name prefix (e.g., `player.rs`, `inventory.rs`).

**Q: Can I use this with monorepos?**
A: Yes! See monorepo examples above for centralized vs distributed patterns.

## Feature Request

Vote for native custom output path support:

**Issue:** getlumos/lumos#XX - Custom output path parameter

Proposed syntax:
```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/*.lumos'
    rust-output: 'src/rust/'      # NEW
    typescript-output: 'src/ts/'  # NEW
```

## See Also

- [Multi-Language Generation](multi-language.md) - Understanding dual-language generation
- [examples/workflows/custom-output-organization.yml](../examples/workflows/custom-output-organization.yml) - Complete examples
- [examples/workflows/monorepo-multi-package.yml](../examples/workflows/monorepo-multi-package.yml) - Monorepo patterns
