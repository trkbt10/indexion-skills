---
name: indexion-plan-docs
description: Analyze documentation coverage and generate documentation plans. Use when the user asks about documentation gaps, wants to improve docs, or needs a documentation roadmap.
---

# indexion plan documentation

Analyze documentation coverage and generate actionable documentation plans.

## When to Use

- User asks "what needs documentation?"
- User wants to check documentation coverage
- User is planning a documentation sprint
- Before a release, to ensure docs are complete
- **After adding new public APIs** — verify doc coverage hasn't dropped

## Usage

```bash
# Full documentation plan
indexion plan documentation <path>

# Coverage analysis only (quick overview — ~8 seconds on large codebases)
indexion plan documentation --style=coverage <path>

# GitHub Issue format for tracking
indexion plan documentation --format=github-issue <path>

# JSON output
indexion plan documentation --format=json <path>

# With GitHub Issue Form template
indexion plan documentation --template=.github/ISSUE_TEMPLATE/doc.yml <path>

# With project name
indexion plan documentation --name=<project> <path>

# Output to file
indexion plan documentation -o=doc-plan.md <path>
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--style=STYLE` | full | Output style: coverage, full |
| `--format=FORMAT` | md | Output format: md, json, github-issue |
| `--template=FILE` | — | GitHub Issue Form template (.yml) |
| `--name=NAME` | auto | Project name (auto-detect from moon.mod.json) |
| `-o, --output=FILE` | stdout | Output file path |
| `--specs-dir=DIR` | kgfs | KGF specs directory |

## How It Works

Public item detection uses **KGF tokenization** (language-agnostic):
- Detects visibility keywords (`pub`, `public`, `export`) followed by declaration
  keywords (`fn`, `struct`, `class`, `enum`, `type`, `trait`, etc.)
- Correctly handles visibility modifiers: `pub(all)`, `pub(readonly)`, `pub(open)`
- Associates `///` doc comments with the declarations they precede
- Applies KGF ignore patterns (e.g. `*_test.mbt`, `*_wbtest.mbt`) to skip test files
- Single-pass tokenization for performance (each file tokenized once, not twice)
- Works for any language with a KGF spec — not hardcoded to any language

## Relationship to grep

`indexion grep --undocumented` provides a quick per-file listing of undocumented
pub declarations. `plan documentation` provides a comprehensive coverage report
with percentages, package breakdown, and prioritized action items.

| Task | Use |
|------|-----|
| "Quick check: what's undocumented?" | `grep --undocumented src/` |
| "Coverage report for release" | `plan documentation --style=coverage .` |
| "Full plan with priorities" | `plan documentation .` |
| "Track in GitHub Issues" | `plan documentation --format=github-issue .` |

## Workflow

1. Run `indexion plan documentation --style=coverage <path>` for a quick overview
2. Run `indexion plan documentation <path>` for a full plan with action items
3. Use `--format=github-issue` to create tracking issues
4. Run `indexion grep --undocumented <path>` during development for quick checks

## Dogfooding Lessons

- `--style=coverage` completes in ~8 seconds on a 750+ file codebase (was hanging
  before performance fix). If it seems stuck, check for test files being included.
- The coverage output shows per-package breakdown — packages with low coverage
  and many pub items are the highest-value documentation targets.
- Coverage can be misleading: `///|` marker-only comments count as documented
  even without descriptive text. Check the `doc_preview` column for quality.
