---
name: indexion-documentation
description: Documentation lifecycle — generate graphs/READMEs, analyze coverage, plan writing tasks, and detect doc drift. Use when the user asks about documentation generation, coverage, staleness, or README planning.
---

# indexion documentation

End-to-end documentation lifecycle: generate, analyze, plan, and verify.

## When to Use

- User asks for a dependency graph or diagram
- User wants to generate README from doc comments
- User asks "what needs documentation?"
- User wants to check documentation coverage
- User asks "are the docs up to date?"
- User wants to create README files for each package
- User asks to plan documentation writing tasks
- Before a release, to ensure docs are complete and up to date

## Commands Overview

| Command | Purpose |
|---------|---------|
| `indexion doc init` | Initialize documentation templates |
| `indexion doc graph` | Generate dependency diagrams |
| `indexion doc readme` | Generate READMEs from source |
| `indexion plan documentation` | Analyze documentation coverage |
| `indexion plan readme` | Generate README writing plans |
| `indexion plan reconcile` | Detect implementation/documentation drift |

Supporting command:

| Command | Purpose |
|---------|---------|
| `indexion grep --undocumented` | Quick per-file listing of undocumented pub declarations |

---

## `indexion doc init` — Initialize Templates

Set up documentation template structure.

```bash
indexion doc init <directory>
indexion doc init --force <directory>
```

**Options:**

| Option | Default | Description |
|--------|---------|-------------|
| `-f, --force` | false | Overwrite existing files |
| `--specs-dir=DIR` | kgfs | KGF specs directory |

Creates `.indexion/state/templates/readme.md` with `{{include:...}}` placeholder directives.

---

## `indexion doc graph` — Dependency Graph

Generate dependency diagrams in various formats.

```bash
# Mermaid (default)
indexion doc graph <path>

# Other formats
indexion doc graph --format=dot <path>
indexion doc graph --format=d2 <path>
indexion doc graph --format=text <path>
indexion doc graph --format=json <path>

# Custom title and output
indexion doc graph --title="My Dependencies" --output=deps.mmd <path>
```

**Options:**

| Option | Default | Description |
|--------|---------|-------------|
| `--format=FORMAT` | mermaid | Output format: mermaid, json, dot, d2, text |
| `--title=TEXT` | Module Dependencies | Diagram title |
| `--output=FILE` | stdout | Output file path |
| `--specs-dir=DIR` | kgfs | KGF specs directory |

---

## `indexion doc readme` — README Generation

Extract doc comments from source files and generate README documentation.

```bash
# Basic
indexion doc readme <path>

# With template
indexion doc readme --template=.indexion/state/templates/readme.md <path>

# With doc.json configuration (root-level README)
indexion doc readme --config=doc.json <path>

# Per-package (skips packages with existing README.md)
indexion doc readme --per-package <path>

# Filter files
indexion doc readme --include='*.mbt' --exclude='*_test.mbt' <path>

# Output to file
indexion doc readme -o=README.md <path>
```

**Options:**

| Option | Default | Description |
|--------|---------|-------------|
| `--format=FORMAT` | md | Output format: markdown, json, raw |
| `--template=FILE` | -- | Template file for generation |
| `--config=FILE` | -- | doc.json configuration file |
| `--per-package` | false | Generate README.md per package (skips existing) |
| `--[no-]recursive` | true | Recurse into subdirectories |
| `--include=PATTERN` | -- | Include pattern (repeatable) |
| `--exclude=PATTERN` | -- | Exclude pattern (repeatable) |
| `--specs-dir=DIR` | kgfs | KGF specs directory |
| `-o, --output=FILE` | stdout | Output file path |

---

## `indexion plan documentation` — Coverage Analysis

Analyze documentation coverage with per-package breakdown and prioritized action items.

```bash
# Full plan (default)
indexion plan documentation <path>

# Quick coverage overview (~8s on large codebases)
indexion plan documentation --style=coverage <path>

# GitHub Issue format
indexion plan documentation --format=github-issue <path>

# JSON output
indexion plan documentation --format=json <path>

# With Issue Form template
indexion plan documentation --template=.github/ISSUE_TEMPLATE/doc.yml <path>

# With project name
indexion plan documentation --name=myproject <path>

# Output to file
indexion plan documentation -o=doc-plan.md <path>
```

**Options:**

| Option | Default | Description |
|--------|---------|-------------|
| `--style=STYLE` | full | Output style: coverage, full |
| `--format=FORMAT` | md | Output format: md, json, github-issue |
| `--template=FILE` | -- | GitHub Issue Form template (.yml) |
| `--name=NAME` | auto | Project name (auto-detect from moon.mod.json) |
| `-o, --output=FILE` | stdout | Output file path |
| `--specs-dir=DIR` | kgfs | KGF specs directory |

### How It Works

Public item detection uses **KGF tokenization** (language-agnostic):
- Detects visibility keywords (`pub`, `public`, `export`) + declaration keywords (`fn`, `struct`, `enum`, `type`, `trait`, etc.)
- Handles visibility modifiers: `pub(all)`, `pub(readonly)`, `pub(open)`
- Associates `///` doc comments with the declarations they precede
- Applies KGF ignore patterns (e.g. `*_test.mbt`, `*_wbtest.mbt`) to skip test files
- Single-pass tokenization for performance

### When to Use Which

| Task | Command |
|------|---------|
| Quick check: what's undocumented? | `indexion grep --undocumented src/` |
| Coverage report for release | `indexion plan documentation --style=coverage .` |
| Full plan with priorities | `indexion plan documentation .` |
| Track in GitHub Issues | `indexion plan documentation --format=github-issue .` |

---

## `indexion plan readme` — README Writing Plans

Analyze templates and generate per-section writing tasks.

```bash
# Generate writing plan from template
indexion plan readme --template=.indexion/state/templates/readme.md <path>

# Output plans to directory
indexion plan readme --template=readme.md --plans-dir=.indexion/plans <path>

# JSON output
indexion plan readme --template=readme.md --format=json <path>

# Output to file
indexion plan readme --template=readme.md -o=readme-plan.md <path>
```

**Options:**

| Option | Default | Description |
|--------|---------|-------------|
| `--template=FILE` | (required) | Template file with `{{include:...}}` placeholders |
| `--plans-dir=DIR` | .indexion/plans | Output directory for plans |
| `--format=FORMAT` | md | Output format: markdown, json |
| `-o, --output=FILE` | stdout | Output file path |
| `--specs-dir=DIR` | kgfs | KGF specs directory |

---

## `indexion plan reconcile` — Documentation Drift Detection

Detect implementation/documentation drift by comparing timestamps and content relationships. Uses inverted-index-accelerated matching and optional fork-based parallelism.

```bash
# Basic check (JSON output)
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

# Custom drift threshold
indexion plan reconcile --threshold-seconds=3600 <path>

# Use git timestamps
indexion plan reconcile --git <path>

# Skip git, use file mtimes only
indexion plan reconcile --mtime-only <path>

# Vocabulary divergence threshold
indexion plan reconcile --vocab-threshold=0.5 <path>

# Limit output
indexion plan reconcile --max-candidates=50 <path>

# Explicit config
indexion plan reconcile --config=.indexion.toml <path>

# GitHub Issue format
indexion plan reconcile --format=github-issue <path>
```

**Options:**

| Option | Default | Description |
|--------|---------|-------------|
| `--format=FORMAT` | json | Output format: json, md, github-issue |
| `--scope=MODE` | -- | Scope mode: custom, package-docs, tree-docs |
| `--specs-dir=DIR` | auto | KGF specs directory |
| `--index-dir=DIR` | .indexion/cache/reconcile | Cache directory |
| `--config=FILE` | -- | Config file (.indexion.toml or .json) |
| `--review-results=FILE` | -- | Apply logical review decisions from JSON |
| `--threshold-seconds=N` | 60 | Drift threshold in seconds |
| `--max-candidates=N` | 200 | Maximum candidates in output |
| `--doc=GLOB` | -- | Restrict document paths (repeatable) |
| `--doc-spec=NAME` | -- | Restrict document KGF specs (repeatable) |
| `--no-file-fallback` | false | Disable basename/file-path fallback matching |
| `--git` | false | Prefer git timestamps over file mtimes |
| `--mtime-only` | false | Skip git timestamps, use file mtimes |
| `--logical-review=MODE` | queue | Logical review mode: queue, off |
| `--vocab-threshold=N` | 0.3 | Vocabulary divergence threshold (cosine distance, 0.0-1.0) |
| `--no-parallel` | false | Disable fork-based parallel processing |
| `-o, --output=FILE` | stdout | Output file path |

---

## End-to-End Workflow

```
1. Initialize    indexion doc init .
2. Understand    indexion doc graph .
3. Assess        indexion plan documentation --style=coverage .
4. Plan          indexion plan readme --template=... .
5. Generate      indexion doc readme --per-package src/ cmd/indexion/
6. Root README   indexion doc readme --config=doc.json .
7. Verify        indexion plan reconcile --format=md .
```

### Configuration

In `.indexion.toml`, the `[doc]` section controls defaults:

```toml
[doc]
config_path = "doc.json"   # doc.json path for root README generation
per_package = true          # Default to per-package mode (no need for --per-package flag)
```

- `per_package = true` makes `indexion doc readme <path>` behave as if `--per-package` was passed.
- Explicit `--config=doc.json` always takes priority for root-level README generation, even when `per_package = true`.

## Dogfooding Lessons

- `doc readme --per-package` skips existing READMEs — hand-written ones are preserved
- `doc graph` output can be embedded directly in README via Mermaid code blocks
- `--style=coverage` completes in ~8s on a 750+ file codebase. If it hangs, check for test files being included.
- Coverage can be misleading: `///|` marker-only comments count as documented even without descriptive text. Check `doc_preview` for quality.
- `plan reconcile` defaults to JSON output — use `--format=md` for human-readable reports
- `plan reconcile` detects **implementation → docs** vocabulary divergence, but does **not** verify the reverse (e.g. READMEs referencing nonexistent CLI options). For that direction, compare each README against `indexion <command> --help` output manually.
- `plan reconcile` cache can become stale after schema changes. If you hit a deserialization crash, clear the cache: `rm -rf .indexion/cache/reconcile`
- Auto-generated READMEs from `doc readme --per-package` are API-listing skeletons. They need manual enrichment with Usage/Options/Examples sections to match actual CLI behavior.
- When verifying README accuracy, always start from the authoritative source: `indexion <command> --help`. READMEs are secondary documentation and drift over time.
- `doc readme --per-package` only generates for packages **without** an existing README. On a mature codebase, nearly all packages already have READMEs (even skeleton ones), so the command skips them. To refresh existing READMEs, edit them manually rather than relying on regeneration.
- The recommended workflow for CLI command READMEs: (1) run `<command> --help`, (2) write Usage/Options/Examples sections from the help output, (3) verify with `plan reconcile --scope=package-docs --format=md` to check vocabulary divergence dropped.
- Root README generation uses `doc.json` to compose sections from package READMEs and static files. Run `indexion doc readme --config=doc.json .` to build it. This is separate from per-package generation.
- `doc.json` entries with nonexistent `path` values are silently ignored. Verify package paths exist before adding them.
