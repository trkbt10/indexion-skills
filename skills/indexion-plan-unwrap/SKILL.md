---
name: indexion-plan-unwrap
description: Detect and remove unnecessary wrapper functions that simply delegate to another function. Use when the user wants to eliminate trivial proxies, reduce indirection, or clean up forwarding functions after a refactor.
---

# indexion plan unwrap

Detect unnecessary wrapper functions and optionally auto-fix them by replacing
callers with direct calls to the delegate.

## When to Use

- User asks to remove unnecessary wrappers or forwarding functions
- After `plan refactor` finds duplicates caused by trivial delegation
- User wants to reduce indirection in the codebase
- User asks "which functions just delegate to another?"
- **After writing new code** — dogfood to catch accidental proxy accumulation

## Relationship to plan refactor

`plan refactor` detects structural similarity between files and functions.
`plan unwrap` complements it by detecting a specific anti-pattern:
functions whose body is a single forwarding call with no added logic.

**Recommended workflow:**
1. `plan unwrap --dry-run` — identify and preview trivial wrappers
2. `plan unwrap --fix` — auto-remove them
3. `plan refactor` — re-run to find remaining real duplicates (with less noise)

## Modes

| Mode | Flag | Description |
|------|------|-------------|
| Report | (default) | List wrappers found (md/json/text) |
| Preview | `--dry-run` | Show all edits: call site replacements + definition deletions |
| Fix | `--fix` | Apply edits to files |

## Usage

```bash
# Report wrappers
indexion plan unwrap --include='*.mbt' <path>

# Preview changes (safe — no files modified)
indexion plan unwrap --dry-run --include='*.mbt' <path>

# Auto-fix: replace call sites and delete wrappers
indexion plan unwrap --fix --include='*.mbt' <path>

# Include self-delegation (self.field.method) and bare constructors
indexion plan unwrap --all --include='*.mbt' <path>

# JSON output
indexion plan unwrap --format=json --include='*.mbt' <path>

# Output to file
indexion plan unwrap -o=unwrap.md --include='*.mbt' <path>
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--dry-run` | — | Preview edits without applying |
| `--fix` | — | Apply edits (replace call sites, delete wrappers) |
| `--all` | — | Include self-delegation and bare constructor wrappers |
| `--include-self` | — | Include `self.field.method` delegation patterns |
| `--include-bare` | — | Include bare constructor wrappers (no qualifier) |
| `--include=PATTERN` | — | Include pattern (repeatable) |
| `--exclude=PATTERN` | — | Exclude pattern (repeatable) |
| `--format=FORMAT` | md | Output format: md, json, text |
| `--output=FILE, -o=` | stdout | Output file path |

## What Gets Detected

A function is flagged as an unwrappable wrapper if:
- Its body is a **single function call**
- Arguments are **simple identifiers** (no expressions)
- No control flow (`if`, `match`, `let`, etc.)

### Detected (default)

```moonbit
fn matches_pattern(text : String, pat : String) -> Bool {
  @glob.glob_match(text, pat)       // trivial delegation
}

fn is_absolute_path(path : String) -> Bool {
  @path_utils.is_absolute(path)     // just a rename
}
```

### Excluded by default (use --all to include)

```moonbit
fn length(self : MyList) -> Int {
  self.items.length()               // self-delegation (encapsulation)
}

fn emit(value : String) -> Action {
  Emit(value)                       // bare constructor
}
```

## Dogfooding Workflow

```bash
# Step 1: Preview wrappers to remove
indexion plan unwrap --dry-run \
  --include='*.mbt' --exclude='*_wbtest.mbt' \
  --exclude='*moon.pkg*' --exclude='*pkg.generated*' \
  src/

# Step 2: Review the output — check each wrapper is truly unnecessary

# Step 3: Apply fixes
indexion plan unwrap --fix \
  --include='*.mbt' --exclude='*_wbtest.mbt' \
  --exclude='*moon.pkg*' --exclude='*pkg.generated*' \
  src/

# Step 4: Run tests to verify
moon test --target native

# Step 5: Re-run plan refactor to see cleaner results
indexion plan refactor --threshold=0.9 \
  --include='*.mbt' --exclude='*_wbtest.mbt' \
  src/
```

## Relationship to grep

`indexion grep --semantic=proxy` detects the same proxy functions as `plan unwrap`,
but without the auto-fix capability. Use `grep` for quick discovery during
development, `plan unwrap` when you want to take action:

```bash
# Quick check: any new proxy functions?
indexion grep --semantic=proxy src/

# Ready to fix: preview and apply
indexion plan unwrap --dry-run --include='*.mbt' src/
indexion plan unwrap --fix --include='*.mbt' src/
```

## Tips

- **Always --dry-run first**: Review what will change before applying `--fix`.
- **Exclude test files**: `--exclude='*_wbtest.mbt'` avoids touching test code.
- **Platform wrappers**: FFI wrappers (e.g. `encode_bytes` → `@utf8.encode`)
  may be intentional abstraction layers. Review before removing.
- **Public API wrappers**: If a wrapper is `pub` and used by external packages,
  removing it is a breaking change. Check visibility before fixing.
