---
name: indexion-plan-readme
description: Generate README documentation writing plans and tasks. Use when the user wants to create README files for packages, plan documentation writing, or generate doc tasks for manual or LLM authoring.
---

# indexion plan readme

Generate README documentation writing plans based on templates and source analysis.

## When to Use

- User wants to create README files for each package
- User asks to plan documentation writing tasks
- User wants to generate doc tasks for manual or LLM authoring
- Before a documentation sprint, to identify what READMEs need writing

## Usage

```bash
# Generate README writing plan using a template
indexion plan readme --template=.indexion/state/templates/readme.md <path>

# Output plans to a directory
indexion plan readme --template=readme.md --plans-dir=.indexion/plans <path>

# JSON output
indexion plan readme --template=readme.md --format=json <path>

# Custom KGF specs directory
indexion plan readme --template=readme.md --specs-dir=kgfs <path>

# Output to file
indexion plan readme --template=readme.md -o=readme-plan.md <path>
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--template=FILE` | (required) | Template file for README generation |
| `--plans-dir=DIR` | .indexion/plans | Output directory for plans |
| `--format=FORMAT` | markdown | Output format: markdown, json |
| `--specs-dir=DIR` | kgfs | KGF specs directory |
| `--output=FILE, -o=` | stdout | Output file path |

## Workflow

1. Run `indexion doc init .` to create a README template
2. Run `indexion plan readme --template=.indexion/state/templates/readme.md .` to generate writing tasks
3. Use the generated plans to write READMEs (manually or with LLM)
