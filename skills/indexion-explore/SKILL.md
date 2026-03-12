---
name: indexion-explore
description: Find similar files, detect duplicates, and analyze code similarity across a codebase. Use when the user asks to find duplicate code, similar files, or wants to understand code overlap.
---

# indexion explore

Analyze file similarity across a directory to find duplicates and related code.

## When to Use

- User asks to find similar or duplicate files
- User wants to understand code overlap before refactoring
- User asks "what files are related to X?"
- User wants to detect copy-paste code

## Usage

```bash
# Basic similarity matrix
indexion explore <path>

# List format with threshold (most useful for finding duplicates)
indexion explore --format=list --threshold=0.7 <path>

# Cluster similar files together
indexion explore --format=cluster --threshold=0.6 <path>

# JSON output for further processing
indexion explore --format=json --threshold=0.5 <path>

# Filter by extension
indexion explore --ext=.mbt --ext=.ts <path>

# Include/exclude patterns
indexion explore --include=*.ts --exclude=*_test.ts src/

# Function-level tree edit distance (more precise, slower)
indexion explore --strategy=apted --format=list <path>
indexion explore --strategy=tsed --format=list <path>
```

## Strategies

| Strategy | Description | Speed |
|----------|-------------|-------|
| `tfidf` (default) | TF-IDF token similarity | Fast |
| `ncd` | Normalized Compression Distance | Fast |
| `hybrid` | Combined NCD + TF-IDF | Fast |
| `apted` | All-Path Tree Edit Distance (function-level) | Slow |
| `tsed` | Tree Structure Edit Distance (function-level) | Slow |

## Output Formats

- `matrix` — Full similarity matrix (default, good for small sets)
- `list` — Sorted pairs above threshold (best for finding duplicates)
- `cluster` — Groups of similar files
- `json` — Machine-readable output

## Workflow

1. Run `indexion explore --format=list --threshold=0.7 <path>` to find similar files
2. Review the pairs and decide which to consolidate
3. Use the results to inform refactoring decisions
