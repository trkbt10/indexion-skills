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
| Scope | Single directory (find internal duplication) | Cross-directory (find overlap between dirs) |
| Goal | Consolidate within a codebase | Extract shared code into a new package |
| Input | One path | `--from=dirA,dirB` (multiple source dirs) |

Use `plan refactor` first to clean up each directory, then `plan solid`
to identify cross-directory extraction candidates.

## Usage

```bash
# Extract common code from multiple source directories
indexion plan solid --from=src/a,src/b <path>

# Specify target directory for extraction
indexion plan solid --from=src/a,src/b --to=src/common <path>

# With rules file
indexion plan solid --from=src/a,src/b --rules=.solidrc <path>

# Inline rules
indexion plan solid --from=src/a,src/b --rule="ignore:*_test.mbt" <path>

# Higher threshold for stricter matching
indexion plan solid --from=src/a,src/b --threshold=0.95 <path>

# Use tree edit distance for precise function-level matching
indexion plan solid --from=src/a,src/b --strategy=apted <path>

# GitHub Issue format
indexion plan solid --from=src/a,src/b --format=github-issue <path>

# Filter files
indexion plan solid --from=src/a,src/b --include=*.mbt --exclude=*_test.mbt <path>

# Output to file
indexion plan solid --from=src/a,src/b -o=solid-plan.md <path>
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--from=DIRS` | (required) | Source directories (comma-separated) |
| `--to=DIR` | — | Target directory for extraction |
| `--rules=FILE` | — | Rules file (.solidrc) |
| `--rule=RULE` | — | Inline rule (repeatable) |
| `--threshold=FLOAT` | 0.9 | Minimum similarity threshold |
| `--strategy=NAME` | tfidf | Similarity strategy: tfidf, apted, tsed |
| `--include=PATTERN` | — | Include pattern (repeatable) |
| `--exclude=PATTERN` | — | Exclude pattern (repeatable) |
| `--format=FORMAT` | md | Output format: md, json, github-issue |
| `--output=FILE, -o=` | stdout | Output file path |

## Workflow

1. Identify directories with suspected duplicate logic
2. Run `indexion plan solid --from=dirA,dirB .` to get a solidification plan
3. Review the identified common code candidates
4. Extract shared code following the plan's recommendations
