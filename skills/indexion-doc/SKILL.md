---
name: indexion-doc
description: Generate dependency graphs and README documentation from source code. Use when the user asks for a dependency diagram, wants to generate docs from code, or needs a project overview.
---

# indexion doc

Generate dependency graphs and README documentation from source code analysis.

## When to Use

- User asks for a dependency graph or diagram
- User wants to generate README from doc comments
- User asks "how are these packages connected?"
- User wants to visualize project structure

## Subcommands

### `indexion doc graph` — Dependency Graph

Generate dependency diagrams in various formats.

```bash
# Mermaid format (default)
indexion doc graph <path>

# Graphviz DOT format
indexion doc graph --format=dot <path>

# D2 diagram format
indexion doc graph --format=d2 <path>

# Plain text
indexion doc graph --format=text <path>

# JSON
indexion doc graph --format=json <path>

# Custom title and output
indexion doc graph --title="My Dependencies" --output=deps.mmd <path>

# Custom KGF specs directory
indexion doc graph --specs=kgfs <path>
```

**Options:**

| Option | Default | Description |
|--------|---------|-------------|
| `--format=FORMAT` | mermaid | Output format: mermaid, json, dot, d2, text |
| `--title=TEXT` | Module Dependencies | Diagram title |
| `--output=FILE` | stdout | Output file path |
| `--specs=DIR` | kgfs | KGF specs directory |

### `indexion doc readme` — README Generation

Generate README.md from source doc comments.

```bash
# Basic README generation
indexion doc readme <path>

# Using a template
indexion doc readme --template=.indexion/state/templates/readme.md <path>

# Using doc.json configuration
indexion doc readme --config=doc.json <path>

# Per-package README generation
indexion doc readme --per-package <path>

# Filter files
indexion doc readme --include=*.mbt --exclude=*_test.mbt <path>

# JSON or raw output
indexion doc readme --format=json <path>
indexion doc readme --format=raw <path>
```

**Options:**

| Option | Default | Description |
|--------|---------|-------------|
| `--format=FORMAT` | markdown | Output format: markdown, json, raw |
| `--template=FILE` | — | Template file for generation |
| `--config=FILE` | — | doc.json configuration file |
| `--per-package` | false | Generate README.md in each package directory |
| `--recursive` | true | Recurse into subdirectories |
| `--no-recursive` | — | Disable recursion |
| `--include=PATTERN` | — | Include pattern (repeatable) |
| `--exclude=PATTERN` | — | Exclude pattern (repeatable) |
| `--specs-dir=DIR` | kgfs | KGF specs directory |
| `--output=FILE, -o=` | stdout | Output file path |

### `indexion doc init` — Initialize Templates

Set up documentation templates.

```bash
# Initialize README template
indexion doc init <path>

# Force overwrite existing
indexion doc init --force <path>
```

## Workflow

1. Run `indexion doc init .` to set up templates
2. Run `indexion doc graph .` to understand dependencies
3. Run `indexion doc readme --template=... .` to generate/update READMEs
