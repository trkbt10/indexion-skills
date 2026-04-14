---
name: indexion-documentation
description: ドキュメントのライフサイクル管理。READMEの自動生成、ドキュメントカバレッジ分析、書くべきドキュメントの計画立案、コード↔ドキュメント間のドリフト検出（plan reconcile）。
---

# indexion documentation — Documentation Lifecycle

Manage documentation from generation through verification. This skill covers
every phase of the documentation lifecycle: assessing what exists, generating
what's missing, detecting when docs drift from code, and planning what to write.

## Start Here: What Do You Need?

### "What needs documentation?"

Assess the current state before writing anything.

```bash
# Quick coverage overview — how much of the public API is documented?
indexion plan documentation --style=coverage .
```

This scans all packages and reports:
- Overall coverage percentage (documented / total pub items)
- Per-package breakdown with README presence
- Functions vs types coverage split

Output example:
```
Overall Coverage: 81% (2285/2806)
Functions: 89%, Types: 75%
```

For a detailed plan with prioritized action items:

```bash
# Full plan with priorities and package inventory
indexion plan documentation .

# As a GitHub Issue for tracking
indexion plan documentation --format=github-issue .

# JSON for scripting
indexion plan documentation --format=json .
```

For a quick per-file listing of undocumented items:

```bash
# Which pub declarations lack doc comments?
indexion grep --undocumented src/
```

**How detection works:** Uses KGF tokenization to find visibility keywords
(`pub`, `public`, `export`) paired with declaration keywords (`fn`, `struct`,
`enum`, `type`, `trait`). Associates `///` doc comments with declarations.
Language-agnostic — works for any KGF-supported language.

**Caveat:** `///|` marker-only comments count as "documented" even without
descriptive text. Check `doc_preview` in the output for quality, not just coverage.

### "Generate READMEs for my packages"

Auto-generate README files from doc comments in source code.

```bash
# Generate README.md for each package that doesn't already have one
indexion doc readme --per-package src/ cmd/indexion/
```

On a mature codebase, most packages already have READMEs, so this command
skips them all. It only creates new files — never overwrites.

For root-level README composed from package READMEs:

```bash
# Build root README from doc.json sections
indexion doc readme --config=doc.json .
```

For a single package output to stdout:

```bash
# Generate README content for a specific path
indexion doc readme src/kgf/lexer/

# With template
indexion doc readme --template=.indexion/state/templates/readme.md src/

# Output to file
indexion doc readme -o=README.md src/kgf/lexer/
```

**Auto-generated READMEs are API-listing skeletons.** They need manual enrichment
with Usage, Options, and Examples sections. For CLI command READMEs, the
authoritative source is `indexion <command> --help`.

### "Are my docs up to date?"

Detect drift between implementation code and documentation.

```bash
# Full reconcile report in markdown
indexion plan reconcile --format=md .
```

This compares code symbols against documentation and reports:
- **Vocabulary divergence**: source code terms missing from co-located docs
- **Stale docs**: code changed after docs were last updated
- **Missing docs**: code modules with no documentation coverage

**Read the report:**

The Vocabulary Divergence table shows distance (0-100%) between code vocabulary
and documentation. 90%+ distance means the README is essentially unrelated to
the current code. Check the Gap Terms column for specific missing vocabulary.

**Scoped checks:**

```bash
# Check only package-level docs
indexion plan reconcile --scope=package-docs .

# Check only tree-level docs
indexion plan reconcile --scope=tree-docs .

# Check specific documents
indexion plan reconcile --doc='docs/**/*.md' .
indexion plan reconcile --doc-spec=markdown .
```

**Timestamp strategies:**

```bash
# Use git commit timestamps (more accurate for collaborative projects)
indexion plan reconcile --git .

# Use file mtimes only (faster, no git dependency)
indexion plan reconcile --mtime-only .
```

**Cache and drift:**

`plan reconcile` maintains a cache at `.indexion/cache/reconcile/`. After schema
changes or indexion upgrades, the cache can become stale and cause deserialization
errors. Clear it:

```bash
rm -rf .indexion/cache/reconcile
```

### "Show me the dependency structure"

Generate dependency diagrams for understanding module relationships.

```bash
# Mermaid diagram (default — embeddable in GitHub README)
indexion doc graph src/config/

# Other formats
indexion doc graph --format=dot src/     # Graphviz DOT
indexion doc graph --format=d2 src/      # D2
indexion doc graph --format=text src/    # ASCII text
indexion doc graph --format=json src/    # Machine-readable

# Custom title and output file
indexion doc graph --title="KGF Dependencies" --output=deps.mmd src/kgf/
```

### "Plan what documentation to write"

Generate structured writing tasks from templates.

```bash
# Initialize documentation template structure
indexion doc init .

# Generate writing plan from template
indexion plan readme --template=.indexion/state/templates/readme.md src/

# Output plans to directory
indexion plan readme --template=readme.md --plans-dir=.indexion/plans src/

# JSON for task tracking
indexion plan readme --template=readme.md --format=json src/
```

## Full Workflow

```bash
# 1. Assess — what's the current state?
indexion plan documentation --style=coverage .

# 2. Generate — create READMEs for packages that lack them
indexion doc readme --per-package src/ cmd/indexion/

# 3. Visualize — understand the dependency structure
indexion doc graph --output=deps.mmd src/

# 4. Enrich — for each CLI command README:
#    a) Run `indexion <command> --help` for the authoritative API
#    b) Write Usage/Options/Examples sections from the help output
#    c) Never copy CLI details from other docs — they drift

# 5. Verify — detect drift between code and docs
indexion plan reconcile --format=md .

# 6. Fix — update docs flagged as divergent, then re-verify
indexion plan reconcile --format=md .
#    → Iterate until vocabulary divergence drops to acceptable levels
```

## Configuration

In `.indexion.toml`, the `[doc]` section controls defaults:

```toml
[doc]
config_path = "doc.json"   # doc.json path for root README generation
per_package = true          # Default to per-package mode
```

- `per_package = true` makes `doc readme <path>` behave as if `--per-package` was passed.
- Explicit `--config=doc.json` always takes priority for root-level README generation.

`doc.json` entries with nonexistent `path` values are silently ignored.
Verify package paths exist before adding them.

## Common Pitfalls

**"doc readme --per-package generated nothing"**
- On a mature codebase, all packages already have READMEs. The command only
  creates new files, never overwrites. To refresh, edit existing READMEs manually.

**"plan reconcile shows 90%+ divergence everywhere"**
- Auto-generated skeleton READMEs (API listing only) have high divergence because
  they lack the vocabulary of the actual implementation. Enrich them with
  descriptions of what the code does, not just what it exports.

**"plan documentation says 100% coverage but docs are wrong"**
- Coverage measures presence of doc comments, not accuracy. A `///|` marker
  counts as documented. Use `plan reconcile` to check content accuracy.

**"plan reconcile crashes on startup"**
- Cache deserialization error after schema changes. Clear it:
  `rm -rf .indexion/cache/reconcile`

**"plan reconcile detects drift I already fixed"**
- The `--git` flag uses commit timestamps. If you fixed docs but haven't committed,
  mtime-based detection (`--mtime-only`) will see the fix, but git-based won't.

**Reconcile only checks implementation → docs direction.** It detects code terms
missing from docs, but does NOT detect docs referencing nonexistent CLI options.
For that direction, compare each README against `indexion <command> --help` manually.
