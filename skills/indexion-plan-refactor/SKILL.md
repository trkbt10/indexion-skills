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
indexion plan refactor --include=*.mbt --exclude=*_test.mbt <path>

# Output to file
indexion plan refactor -o=refactor-plan.md <path>
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--threshold=FLOAT` | 0.7 | Minimum similarity threshold |
| `--strategy=NAME` | tfidf | Similarity strategy: hybrid, ncd, tfidf |
| `--style=STYLE` | raw | Output style: raw, structured |
| `--format=FORMAT` | md | Output format: md, json, text, github-issue |
| `--name=NAME` | — | Project name (for structured style) |
| `--include=PATTERN` | — | Include pattern (repeatable) |
| `--exclude=PATTERN` | — | Exclude pattern (repeatable) |
| `--output=FILE, -o=` | stdout | Output file path |

## Dogfooding Workflow (Proven)

This workflow was validated by using indexion on itself to eliminate 53+ duplicate
utility functions across 12 CLI packages.

### Step 1: Detect duplicates

```bash
# Find code duplicates above 90% (high-confidence matches)
indexion plan refactor --threshold=0.9 \
  --include='*.mbt' --exclude='*_wbtest.mbt' \
  --exclude='*moon.pkg*' --exclude='*pkg.generated*' \
  cmd/indexion/
```

### Step 2: Analyze the output

Focus on the **Duplicate Code Blocks** section. Key patterns to look for:
- **Identical utility functions** (e.g., `trim_string`, `parse_double`) → extract to a shared module
- **Same logic with different names** (e.g., `format_refactor_issue` vs `format_github_issue`) → unify
- **Hardcoded patterns** (e.g., `has_prefix("pub fn ")`) → replace with data-driven approach
- **CLI boilerplate** (e.g., `--output=`/`-o=` parsing) → extract common parser

### Step 3: Fix and verify

After consolidating duplicates, re-run with the same threshold to confirm
they no longer appear. Lower the threshold to find more candidates:

```bash
# After fixing 90%+ duplicates, look for 85%+ matches
indexion plan refactor --threshold=0.85 --include='*.mbt' ...
```

### Step 4: Iterate

Repeat steps 1-3 with progressively lower thresholds until only
structural patterns remain (CLI boilerplate, doc comment headers).

## Tips

- **Exclude noise**: Always exclude `*_wbtest.mbt`, `*moon.pkg*`, `*pkg.generated*`
  to focus on actual code duplication rather than config/test similarity.
- **Start high**: Begin with `--threshold=0.9` and work down. Low thresholds
  produce huge result sets that are hard to act on.
- **Combine with explore**: Use `indexion explore --format=list` first to get
  a quick similarity overview, then use `plan refactor` for actionable detail.
- **Per-pair detail**: The output shows exact line numbers and function names
  for each duplicate block — use these to navigate directly to the code.
