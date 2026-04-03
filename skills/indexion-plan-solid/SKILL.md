---
name: indexion-plan-solid
description: Generate solidification plans for extracting common code into shared packages. Use when the user wants to extract shared logic, consolidate code across directories, or plan a common library extraction.
---

# indexion plan solid

Generate solidification plans for common code extraction across directories.

## When to Use

- User wants to extract shared code into a common package
- User asks "what code is duplicated between these directories?"
- Planning a shared library extraction from multiple packages
- User wants to consolidate similar implementations

## Difference from plan refactor

| | `plan refactor` | `plan solid` |
|---|-----------------|-------------|
| Scope | Single or multiple directories (find internal duplication) | Cross-directory (find overlap between specific dirs) |
| Goal | Consolidate within a codebase | Extract shared code into a new package |
| Input | `<path>` | `--from=dirA,dirB` (multiple source dirs) |

Use `plan refactor` first to clean up each directory, then `plan solid`
to identify cross-directory extraction candidates.

## Usage

```bash
# Extract common code from multiple source directories
indexion plan solid --from=src/a,src/b

# Specify target directory for extraction
indexion plan solid --from=src/a,src/b --to=src/common

# With rules file
indexion plan solid --from=src/a,src/b --rules=.solidrc

# Inline rules
indexion plan solid --from=src/a,src/b --rule="ignore:*_test.mbt"

# Higher threshold for stricter matching
indexion plan solid --from=src/a,src/b --threshold=0.95

# Use tree edit distance for precise function-level matching
indexion plan solid --from=src/a,src/b --strategy=apted

# GitHub Issue format
indexion plan solid --from=src/a,src/b --format=github-issue

# Filter files
indexion plan solid --from=src/a,src/b --include='*.mbt' --exclude='*_test.mbt'

# Output to file
indexion plan solid --from=src/a,src/b -o=solid-plan.md
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--from=DIRS` | (required) | Source directories (comma-separated or repeatable) |
| `--to=DIR` | — | Target directory for extraction |
| `--rules=FILE` | — | Rules file (.solidrc) |
| `--rule=RULE` | — | Inline rule (repeatable) |
| `--threshold=FLOAT` | 0.9 | Minimum similarity threshold |
| `--strategy=NAME` | tfidf | Similarity strategy: tfidf, apted, tsed |
| `--include=PATTERN` | — | Include pattern (repeatable) |
| `--exclude=PATTERN` | — | Exclude pattern (repeatable) |
| `--format=FORMAT` | md | Output format: md, json, github-issue |
| `-o, --output=FILE` | stdout | Output file path |
| `--specs-dir=DIR` | kgfs | KGF specs directory |

## Workflow

1. Identify directories with suspected duplicate logic
2. Run `indexion plan solid --from=dirA,dirB` to get a solidification plan
3. Review the identified common code candidates
4. Extract shared code following the plan's recommendations
5. Use `indexion grep "TypeIdent:SharedType"` to verify all references are updated

## Dogfooding Lessons

- Run `plan refactor` on each directory individually first to clean internal
  duplication, then `plan solid` across directories for cleaner cross-dir results.
- The `--from` flag accepts comma-separated dirs or can be repeated:
  `--from=src/a --from=src/b` is equivalent to `--from=src/a,src/b`.
