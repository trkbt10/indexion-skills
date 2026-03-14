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
- **Quick scan before detailed refactoring** — use as a first pass, then
  follow up with `indexion plan refactor` for actionable detail

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

# Filter out config noise
indexion explore --format=list --threshold=0.7 \
  --include='*.mbt' --exclude='*moon.pkg*' cmd/indexion/

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

## Workflow: explore → plan refactor

`explore` and `plan refactor` are complementary:

| | `explore` | `plan refactor` |
|---|-----------|-----------------|
| Speed | Fast overview | Deeper analysis |
| Output | Similarity pairs | Duplicate blocks with line numbers |
| Use | "What's similar?" | "What exactly is duplicated and how to fix it?" |

1. Run `indexion explore --format=list --threshold=0.7 <path>` for a quick scan
2. If high-similarity pairs exist, run `indexion plan refactor --threshold=0.9 <path>` for details
3. Fix duplicates, then re-run both to verify

## Tips

- **moon.pkg files** inflate similarity scores (they all look alike) — exclude
  with `--exclude='*moon.pkg*'` for meaningful results
- **96%+ similarity** between CLI files usually means duplicated utility functions
- **85-95% similarity** is often structural (same CLI patterns) — not always actionable
