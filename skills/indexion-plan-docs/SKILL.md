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

# Coverage analysis only (quick overview)
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
| `--output=FILE, -o=` | stdout | Output file path |

## How It Works

Public item detection uses **KGF tokenization** (language-agnostic):
- Detects visibility keywords (`pub`, `public`, `export`) followed by declaration
  keywords (`fn`, `struct`, `class`, `enum`, `type`, `trait`, etc.)
- Associates `///` doc comments with the declarations they precede
- Works for any language with a KGF spec — not hardcoded to MoonBit

## Workflow

1. Run `indexion plan documentation --style=coverage <path>` for a quick overview
2. Run `indexion plan documentation <path>` for a full plan with action items
3. Use `--format=github-issue` to create tracking issues
