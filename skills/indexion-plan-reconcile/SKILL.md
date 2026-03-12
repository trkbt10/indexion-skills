---
name: indexion-plan-reconcile
description: Detect drift between implementation and documentation. Use when the user suspects docs are outdated, wants to find stale documentation, or needs to verify docs match the code.
---

# indexion plan reconcile

Detect implementation/documentation drift by comparing file timestamps and content relationships.

## When to Use

- User asks "are the docs up to date?"
- User suspects documentation is stale
- Before a release, to verify docs match code
- After a large refactor, to find docs that need updating

## Usage

```bash
# Basic reconciliation check
indexion plan reconcile <path>

# Markdown report
indexion plan reconcile --format=md <path>

# Scoped checks
indexion plan reconcile --scope=package-docs <path>
indexion plan reconcile --scope=tree-docs <path>

# Restrict to specific document paths
indexion plan reconcile --doc='docs/**/*.md' <path>

# Restrict to specific document types (KGF spec names)
indexion plan reconcile --doc-spec=markdown <path>

# Custom drift threshold (seconds)
indexion plan reconcile --threshold-seconds=3600 <path>

# Skip git timestamps, use file mtimes only
indexion plan reconcile --mtime-only <path>

# Limit output
indexion plan reconcile --max-candidates=50 <path>

# Use explicit config
indexion plan reconcile --config=.indexion.toml <path>

# GitHub Issue format
indexion plan reconcile --format=github-issue <path>
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--format=FORMAT` | json | Output format: json, md, github-issue |
| `--scope=MODE` | — | Scope mode: custom, package-docs, tree-docs |
| `--specs=DIR` | auto | KGF specs directory |
| `--index-dir=DIR` | .indexion/cache/reconcile | Cache directory |
| `--config=FILE` | — | Config file (.indexion.toml or .json) |
| `--review-results=FILE` | — | Apply logical review decisions from JSON |
| `--threshold-seconds=N` | 60 | Drift threshold in seconds |
| `--max-candidates=N` | 200 | Maximum candidates in output |
| `--doc=GLOB` | — | Restrict document paths (repeatable) |
| `--doc-spec=NAME` | — | Restrict document KGF specs (repeatable) |
| `--no-file-fallback` | false | Disable basename/file-path fallback matching |
| `--mtime-only` | false | Skip git timestamps, use file mtimes |
| `--logical-review=MODE` | queue | Logical review mode: queue, off |
| `--output=FILE, -o=` | stdout | Output file path |

## Workflow

1. Run `indexion plan reconcile .` to scan for drift
2. Review candidates — each shows the source file and its related docs
3. Update stale documentation
4. Re-run to verify all drift is resolved
