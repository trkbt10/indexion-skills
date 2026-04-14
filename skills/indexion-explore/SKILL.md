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
# Basic similarity matrix (default: hybrid strategy)
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
indexion explore --include='*.ts' --exclude='*_test.ts' src/

# Filter out config noise
indexion explore --format=list --threshold=0.7 \
  --include='*.mbt' --exclude='*moon.pkg*' cmd/indexion/

# Function-level tree edit distance (more precise, slower)
indexion explore --strategy=apted --format=list <path>
indexion explore --strategy=tsed --format=list <path>

# Explicit hybrid (default, can be omitted)
indexion explore --strategy=hybrid --format=list <path>
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--format=FORMAT` | matrix | Output: matrix, list, cluster, json |
| `--strategy=NAME` | hybrid | Algorithm: hybrid, tfidf, bm25, jsd, ncd, apted, tsed |
| `--threshold=FLOAT` | 0.5 | Min similarity for list/json/cluster output |
| `--ext=EXT` | all | File extension filter (repeatable) |
| `--include=PATTERN` | -- | Include files matching glob pattern (repeatable) |
| `--exclude=PATTERN` | -- | Exclude files matching glob pattern (repeatable) |
| `--fdr=FLOAT` | 0 | FDR correction level (0=disabled, 0.05=5% false discovery rate) |
| `--specs-dir=DIR` | kgfs | KGF specs directory |

## Strategies

| Strategy | Description | Speed |
|----------|-------------|-------|
| `hybrid` (default) | Multi-signal evidence fusion: BM25 + JSD lexical, TSED structural | Adaptive |
| `tfidf` | TF-IDF token similarity | Fast |
| `bm25` | BM25 token similarity | Fast |
| `jsd` | Jensen-Shannon Divergence | Fast |
| `ncd` | Normalized Compression Distance | Fast |
| `apted` | All-Path Tree Edit Distance (function-level) | Slow |
| `tsed` | Tree Structure Edit Distance (function-level) | Slow |

## Output Formats

- `matrix` — Full similarity matrix (default, good for small sets)
- `list` — Sorted pairs above threshold (best for finding duplicates)
- `cluster` — Groups of similar files
- `json` — Machine-readable output

## Relationship to Other Commands

| Task | Use |
|------|-----|
| "What files are similar?" | `explore --format=list` |
| "Find nested for loops" | `grep "for ... for"` |
| "Find functions named sort" | `grep --semantic=name:sort` |
| "What exactly is duplicated?" | `plan refactor --threshold=0.9` |
| "Find code similar to a description" | `grep --semantic="similar:..."` |

## Workflow: explore → plan refactor

1. Run `indexion explore --format=list --threshold=0.7 <path>` for a quick scan
2. If high-similarity pairs exist, run `indexion plan refactor --threshold=0.9 <path>` for details
3. Fix duplicates, then re-run both to verify

## Dogfooding Lessons

- **moon.pkg files** inflate similarity scores (they all look alike) — exclude
  with `--exclude='*moon.pkg*'` for meaningful results
- **96%+ similarity** between CLI files usually means duplicated utility functions
- **85-95% similarity** is often structural (same CLI patterns) — not always actionable
- **types.mbt files** showing 100% similarity is normal — type definition files
  share structural patterns (pub struct + getters) that inflate TF-IDF scores
