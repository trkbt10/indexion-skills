---
name: indexion-readme
description: README construction — initialize template structure, generate per-package READMEs from doc comments, plan writing tasks, and assemble root README from docs/ and package READMEs via doc.json config.
---

# indexion readme — README Construction

Build project READMEs from templates, doc comments, and hand-written prose.
This skill covers the **construction side**: scaffolding, generating, planning,
and assembling. For evaluating existing documentation, see `indexion-documentation`.

## Workflow Overview

```
doc init               → .indexion/readme/template.md + doc.json
doc readme --per-package → src/*/README.md (API skeletons)
(hand-write)           → docs/intro.md (project prose)
plan readme            → writing plans per section
doc readme --config    → README.md (assembled from all of the above)
```

## Step 1: Initialize

```bash
indexion doc init <project-dir>
```

Creates `.indexion/readme/` with two files:

| File | Purpose |
|------|---------|
| `template.md` | README template with `{{include:...}}` placeholders |
| `doc.json` | Config: which sections, which packages, where to output |

Both files use SoT constants — the paths are consistent across all commands.

## Step 2: Configure doc.json

Edit `.indexion/readme/doc.json` to declare your project structure:

```json
{
  "root": {
    "template": ".indexion/readme/template.md",
    "output": "README.md",
    "sections": [
      { "type": "static", "file": "docs/intro.md" },
      { "type": "toc", "title": "Packages" },
      { "type": "packages", "filter": "src/**" }
    ]
  },
  "packages": [
    {
      "path": "src/objects",
      "title": "objects",
      "description": "Core types",
      "include_in_root": true,
      "sections": ["API"]
    }
  ]
}
```

**Section types:**
- `static` — include a markdown file verbatim
- `toc` — insert a table of contents heading
- `packages` — pull in filtered package READMEs

**Package fields:**
- `include_in_root` — whether to include in the assembled root README
- `sections` — which README sections to pull (e.g. `["API", "Usage"]`)

## Step 3: Generate per-package READMEs

```bash
# Generate README.md for each package that doesn't already have one
indexion doc readme --per-package src/ cmd/
```

These are **API-listing skeletons** extracted from doc comments via KGF.
They only create new files — never overwrite existing ones.

For a single package to stdout:

```bash
indexion doc readme src/kgf/lexer/
indexion doc readme -o=README.md src/kgf/lexer/
```

## Step 4: Write static content

Create `docs/intro.md` (and other files referenced by the template).
This is the hand-written prose — features, architecture, usage examples.
The template's `{{include:docs/intro.md}}` pulls it in during assembly.

## Step 5: Generate writing plans (optional)

```bash
# Get section-by-section writing guidance
indexion plan readme --template=.indexion/readme/template.md src/

# Output plans to a directory
indexion plan readme --template=.indexion/readme/template.md --plans-dir=.indexion/plans src/
```

## Step 6: Assemble the README

Two equivalent approaches:

```bash
# Config-based (recommended): doc.json controls everything
indexion doc readme --config=.indexion/readme/doc.json

# Template-based: {{include:...}} placeholders pull in docs/ files
indexion doc readme --template=.indexion/readme/template.md -o=README.md src/
```

The config-based approach resolves all relative paths in doc.json relative
to the project root (detected via `.indexion/` directory), so it works
correctly even when invoked from a different working directory.

## Template Syntax

The template file supports `{{placeholder}}` substitution:

| Placeholder | Expansion |
|-------------|-----------|
| `{{include:path}}` | Contents of the file (relative to project root) |
| `{{packages}}` | All discovered packages (filtered by CLI --include/--exclude) |
| `{{module_doc}}` | Module-level documentation only |

## .indexion.toml integration

```toml
[doc]
config_path = ".indexion/readme/doc.json"
per_package = true
```

- `config_path` — auto-loads doc.json without `--config` flag
- `per_package = true` — makes `doc readme <path>` default to `--per-package`
- Explicit `--config=...` always takes priority

## Common Pitfalls

**"doc readme --per-package generated nothing"**
- All packages already have READMEs. The command only creates new files,
  never overwrites. Delete existing READMEs first to regenerate.

**"doc readme --config assembled wrong files"**
- Relative paths in doc.json resolve relative to the project root (detected
  via project root markers). If no project root is found, paths resolve
  relative to the doc.json file's parent directory.

**"Auto-generated READMEs are just API listings"**
- By design. Per-package READMEs are skeletons. Enrich them manually with
  Usage, Options, and Examples sections. For CLI commands, the authoritative
  source is `indexion <command> --help`.

**"Assembled README is thousands of lines long"**
- The `packages` section pulls full API listings from each package README.
  For large projects, either remove the `{ "type": "packages" }` section
  from doc.json and write a manual package summary table in `docs/intro.md`,
  or limit which packages have `include_in_root: true` in doc.json.
