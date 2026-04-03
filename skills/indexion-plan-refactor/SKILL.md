---
name: indexion-plan-refactor
description: Generate refactoring plans based on code similarity analysis. Use when the user asks to plan a refactor, find code to consolidate, or reduce duplication across a codebase.
---

# indexion plan refactor

Generate structured refactoring plans from file similarity analysis.

## When to Use

- User asks to plan a refactoring
- User wants to reduce code duplication
- User asks "what should I consolidate?"
- Before a large-scale code cleanup
- **After writing new code** — dogfood to catch accidental duplication

## Usage

```bash
# Basic refactoring plan
indexion plan refactor <path>

# With higher similarity threshold (stricter matches)
indexion plan refactor --threshold=0.8 <path>

# Structured plan with phases and checklists
indexion plan refactor --style=structured --name=<project> <path>

# Choose similarity strategy
indexion plan refactor --strategy=tfidf <path>
indexion plan refactor --strategy=ncd <path>
indexion plan refactor --strategy=hybrid <path>

# GitHub Issue format
indexion plan refactor --format=github-issue <path>

# Filter files
indexion plan refactor --include='*.mbt' --exclude='*_test.mbt' <path>

# Output to file
indexion plan refactor -o=refactor-plan.md <path>

# Multiple directories
indexion plan refactor src/ cmd/indexion/
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--threshold=FLOAT` | 0.7 | Minimum similarity threshold |
| `--strategy=NAME` | hybrid | Similarity strategy: hybrid, tfidf, ncd |
| `--style=STYLE` | raw | Output style: raw, structured |
| `--format=FORMAT` | md | Output format: md, json, text, github-issue |
| `--name=NAME` | — | Project name (for structured style) |
| `--include=PATTERN` | — | Include pattern (repeatable) |
| `--exclude=PATTERN` | — | Exclude pattern (repeatable) |
| `-o, --output=FILE` | stdout | Output file path |
| `--specs-dir=DIR` | kgfs | KGF specs directory |

## Dogfooding Workflow (Proven)

This workflow was validated by using indexion on itself to eliminate 10+
duplicate utility functions and resolve a circular dependency.

### Step 1: Detect duplicates

```bash
# Find code duplicates above 90% (high-confidence matches)
indexion plan refactor --threshold=0.9 \
  --include='*.mbt' --exclude='*_wbtest.mbt' \
  --exclude='*moon.pkg*' --exclude='*pkg.generated*' \
  cmd/indexion/
```

### Step 2: Analyze the output

Focus on **three sections**:

**Duplicate Code Blocks** — line-level identical code between files:
- Identical utility functions → extract to `@common` or shared module
- Same logic with different names → unify

**Function-Level Duplicates** — structurally similar functions (TF-IDF on function bodies):
- Cross-file: same function copied between packages → extract to SoT
- Same-file: nearly identical functions → parameterize into one

**Same-file duplicates** — functions within one file that are 90%+ similar:
- These are the highest-value targets (easiest to fix, clearest wins)
- Example: `get_global_data_dir ≈ get_global_cache_dir` → extract `resolve_os_dir`

### Step 3: Use grep to trace references before fixing

```bash
# Find all references to a type/function before moving it
indexion grep "TypeIdent:TfidfEmbeddingProvider" src/

# Find all functions named "substr" to plan consolidation
indexion grep --semantic=name:substr src/
```

### Step 4: Fix and verify

After consolidating duplicates, re-run with the same threshold to confirm
they no longer appear:

```bash
indexion plan refactor --threshold=0.9 --include='*.mbt' ...
```

### Step 5: Lower threshold and iterate

```bash
# After fixing 90%+ duplicates, look for 85%+ matches
indexion plan refactor --threshold=0.85 --include='*.mbt' ...
```

## What Remains After Cleanup

When all actionable duplicates are fixed, you'll see only these in the output:
- **Platform stubs** (`native.mbt` / `stub.mbt`) — intentional platform branching
- **Type method similarity** (`to_string` on different types) — different types, same pattern
- **CLI command boilerplate** (`command()` functions) — @argparse API pattern, not duplication
- **Semantic-but-different** functions (`is_disqualifying_keyword` ≈ `is_skip_token`) — different purpose

These are signals to stop iterating, not bugs to fix.

## Tips

- **Exclude noise**: Always exclude `*_wbtest.mbt`, `*moon.pkg*`, `*pkg.generated*`
  to focus on actual code duplication rather than config/test similarity.
- **Start high**: Begin with `--threshold=0.9` and work down. Low thresholds
  produce huge result sets that are hard to act on.
- **Use grep first**: `indexion grep --semantic=name:X` finds all functions with
  a specific name. `grep --semantic=proxy` finds wrappers. Use for targeted
  investigation before running the full plan.
- **Circular dependencies**: If `plan refactor` shows cross-package duplication
  caused by circular imports, use `indexion grep "TypeIdent:X"` to map all
  references, then move the type to break the cycle (as was done with `Sentence`
  → `segmentation/types`).
- **Combine with explore**: Use `indexion explore --format=list` first to get
  a quick similarity overview, then use `plan refactor` for actionable detail.
