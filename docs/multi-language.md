# Multi-Language Generation

## Overview

LUMOS **always generates both Rust and TypeScript** code from a single schema. This is by design to ensure type-safe interoperability between Solana programs (Rust) and clients (TypeScript).

## How LUMOS Generates Code

When you run the action:

```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/player.lumos'
```

LUMOS generates **two files** in the same directory as the schema:

```
schemas/
├── player.lumos      # Your schema
├── generated.rs      # Rust code (Anchor/Borsh)
└── generated.ts      # TypeScript code + Borsh schemas
```

**You cannot generate only Rust or only TypeScript.** Both languages are generated simultaneously to guarantee serialization compatibility.

## Why Both Languages?

LUMOS ensures type safety across the entire stack:

1. **Rust** - Solana program structs with Borsh serialization
2. **TypeScript** - Client-side interfaces with matching Borsh schemas
3. **Guaranteed compatibility** - Same schema = same serialization

This prevents runtime errors from type mismatches between program and client.

## Current Limitation

**The action does NOT support:**
- Generating only Rust code
- Generating only TypeScript code
- Separate output paths per language

**Workaround:** Use file organization strategies (see below).

## Workaround: Organize Outputs by Language

If you need separate directories for Rust and TypeScript, use post-generation file organization:

### Strategy 1: Copy to Language-Specific Directories

```yaml
name: Separate Language Outputs

on: [push, pull_request]

jobs:
  generate-and-organize:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Generate both languages
      - uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          version: 'latest'

      # Organize Rust files
      - name: Move Rust to dedicated directory
        run: |
          mkdir -p packages/rust-generated
          find schemas -name "generated.rs" -exec cp {} packages/rust-generated/ \;

      # Organize TypeScript files
      - name: Move TypeScript to dedicated directory
        run: |
          mkdir -p packages/ts-generated
          find schemas -name "generated.ts" -exec cp {} packages/ts-generated/ \;

      - name: Commit organized files
        if: github.ref == 'refs/heads/main'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add packages/
          git commit -m "chore: Update generated code [skip ci]" || exit 0
          git push
```

### Strategy 2: Symbolic Links

Keep generated files in place, create symlinks in language directories:

```yaml
- name: Create symlinks to generated code
  run: |
    mkdir -p src/rust src/typescript

    # Link Rust files
    find schemas -name "generated.rs" | while read file; do
      ln -sf "$(realpath $file)" src/rust/
    done

    # Link TypeScript files
    find schemas -name "generated.ts" | while read file; do
      ln -sf "$(realpath $file)" src/typescript/
    done
```

### Strategy 3: Rename and Move

Rename files with schema name prefix:

```yaml
- name: Rename and organize by schema name
  run: |
    mkdir -p generated/{rust,typescript}

    # Find all schema files
    find schemas -name "*.lumos" | while read schema; do
      schema_name=$(basename "$schema" .lumos)
      schema_dir=$(dirname "$schema")

      # Move and rename Rust file
      if [ -f "$schema_dir/generated.rs" ]; then
        mv "$schema_dir/generated.rs" \
           "generated/rust/${schema_name}.rs"
      fi

      # Move and rename TypeScript file
      if [ -f "$schema_dir/generated.ts" ]; then
        mv "$schema_dir/generated.ts" \
           "generated/typescript/${schema_name}.ts"
      fi
    done
```

## Multi-Package Generation

For monorepos with multiple packages, use GitHub Actions matrix strategy:

```yaml
name: Multi-Package Generation

on: [push, pull_request]

jobs:
  generate-all-packages:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        package: [auth, payments, users, analytics]
    steps:
      - uses: actions/checkout@v4

      - name: Generate ${{ matrix.package }} schemas
        uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'

      - name: Organize package outputs
        run: |
          cd packages/${{ matrix.package }}
          mkdir -p generated/rust generated/typescript

          # Move to organized directories
          find schemas -name "generated.rs" \
            -exec mv {} generated/rust/ \; 2>/dev/null || true
          find schemas -name "generated.ts" \
            -exec mv {} generated/typescript/ \; 2>/dev/null || true
```

**Benefits:**
- ✅ Each package validated independently
- ✅ Parallel execution (faster)
- ✅ Failures isolated per package
- ✅ Clean organization

## Monorepo Example: Centralized vs Distributed

### Distributed (Each Package Has Own Generated Code)

```
packages/
├── auth/
│   ├── schemas/
│   │   └── user.lumos
│   └── generated/
│       ├── rust/
│       │   └── generated.rs
│       └── typescript/
│           └── generated.ts
├── payments/
│   ├── schemas/
│   │   └── transaction.lumos
│   └── generated/
│       ├── rust/
│       │   └── generated.rs
│       └── typescript/
│           └── generated.ts
```

**Workflow:**
```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'packages/*/schemas/*.lumos'

- name: Organize each package
  run: |
    for pkg in packages/*/; do
      cd "$pkg"
      mkdir -p generated/rust generated/typescript
      mv schemas/generated.rs generated/rust/ 2>/dev/null || true
      mv schemas/generated.ts generated/typescript/ 2>/dev/null || true
      cd ../..
    done
```

### Centralized (All Generated Code in One Location)

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
packages/
├── auth/schemas/user.lumos
├── payments/schemas/transaction.lumos
└── users/schemas/profile.lumos
```

**Workflow:**
```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'packages/*/schemas/*.lumos'

- name: Centralize all generated code
  run: |
    mkdir -p generated/{rust,typescript}

    # Collect all Rust files
    find packages -name "generated.rs" | while read file; do
      pkg=$(echo "$file" | cut -d'/' -f2)
      mv "$file" "generated/rust/${pkg}.rs"
    done

    # Collect all TypeScript files
    find packages -name "generated.ts" | while read file; do
      pkg=$(echo "$file" | cut -d'/' -f2)
      mv "$file" "generated/typescript/${pkg}.ts"
    done
```

## FAQ

**Q: Can I generate only Rust code?**
A: No, LUMOS always generates both languages. Use file organization to separate them after generation.

**Q: Why generate both languages?**
A: To guarantee type-safe Borsh serialization compatibility between Solana programs and clients.

**Q: Can I customize the output filenames?**
A: Not directly. LUMOS always generates `generated.rs` and `generated.ts`. Use post-generation renaming.

**Q: Does this work with multiple schemas?**
A: Yes! Each schema generates its own `generated.rs` and `generated.ts` pair in the schema's directory.

**Q: What if I only use Rust?**
A: TypeScript files will still be generated. You can ignore or delete them (though they're useful for documentation).

## Feature Request

Want native support for separate language outputs? Vote on the feature request:

**Issue:** getlumos/lumos#XX - Separate language generation support

Proposed syntax:
```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/*.lumos'
    languages: 'rust,typescript'  # NEW: Optional language selection
    rust-output: 'src/rust/'      # NEW: Custom Rust path
    typescript-output: 'src/ts/'  # NEW: Custom TypeScript path
```

## See Also

- [Custom Output Paths](custom-output-paths.md) - Workarounds for custom output locations
- [examples/workflows/separate-rust-typescript.yml](../examples/workflows/separate-rust-typescript.yml) - Complete example
- [examples/workflows/monorepo-multi-package.yml](../examples/workflows/monorepo-multi-package.yml) - Matrix strategy
