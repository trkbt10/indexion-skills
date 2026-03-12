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

## Workflow

1. Run `indexion plan refactor --style=structured <path>` to get a full plan
2. Review the identified similarity groups
3. Prioritize based on the plan's recommendations
4. Execute refactoring following the suggested phases
