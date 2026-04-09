---
name: indexion-wiki
description: Build, maintain, and verify a project wiki using indexion. Use when the user wants to create wiki pages, update stale documentation, check wiki health, or navigate the wiki as an agent.
---

# indexion wiki

indexion's wiki system maintains a knowledge base at `.indexion/wiki/` as a collection of
Markdown files governed by a `wiki.json` manifest. The system is designed for LLM-agent
collaboration: tools detect what needs to change; agents do the writing.

## The Three-Layer Model

```
Source Files (immutable) → Wiki Pages (agent-maintained) → Structural Checks (deterministic)
```

- **Source files** are the ground truth. The wiki synthesizes knowledge from them.
- **Wiki pages** are Markdown files + a manifest (`wiki.json`) tracking provenance and authorship.
- **Structural checks** (`wiki lint`) run without an LLM and tell you when the wiki is inconsistent.

Every wiki page carries two provenance fields:
- `provenance`: `"extracted"` | `"synthesized"` | `"manual"`
- `last_actor`: `"indexion"` | `"agent:<name>"` | `"user"`

## Command Structure

```
indexion wiki
  ├── pages
  │   ├── plan    -- propose page structure (init-like)
  │   ├── add     -- add a new page
  │   ├── update  -- update an existing page
  │   └── ingest  -- detect stale pages
  ├── index
  │   └── build   -- build index.md and optionally vectors.db
  ├── lint        -- structural integrity checks
  ├── export      -- export to GitHub/GitLab wiki format
  ├── import      -- import from GitHub/GitLab wiki format
  └── log         -- operation audit trail
```

## Navigating the Wiki (as an agent)

Before reading individual pages, start with the index:

```bash
# Generate or regenerate index.md — the entry point for navigation
indexion wiki index build --wiki-dir=.indexion/wiki

# Then read it
cat .indexion/wiki/index.md
```

`index.md` lists pages by category and identifies **hub pages** (most-linked pages).
Hub pages are the best starting points for understanding an unfamiliar codebase.

## Writing a New Page

```bash
# 1. Write content to a temp file
cat > /tmp/my-page.md << 'EOF'
# My Page Title
...content...
EOF

# 2. Register it in the manifest and write the .md file
indexion wiki pages add \
  --id=my-page \
  --title="My Page Title" \
  --content=/tmp/my-page.md \
  --sources="src/my-module/" \
  --provenance=synthesized \
  --actor="agent:claude" \
  --wiki-dir=.indexion/wiki
```

The `--sources` field links the page to source files for change tracking.
Always specify it — pages without sources are invisible to `wiki pages ingest`.

## Updating an Existing Page

```bash
# Write new content to a temp file, then:
indexion wiki pages update \
  --id=existing-page \
  --content=/tmp/updated.md \
  --sources="src/my-module/,src/other/" \
  --provenance=synthesized \
  --actor="agent:claude" \
  --wiki-dir=.indexion/wiki
```

## Detecting What Needs Updating

```bash
# Find pages whose source files have changed since the last run
indexion wiki pages ingest --wiki-dir=.indexion/wiki

# Preview without recording the new hash state
indexion wiki pages ingest --dry-run --wiki-dir=.indexion/wiki
```

`ingest` hashes every source file listed in `wiki.json` and compares against
`.indexion/wiki/ingest-manifest.json`. It outputs an update task list but does
**not** rewrite pages — that is the agent's responsibility.

Workflow:
1. Run `wiki pages ingest` to get the task list.
2. For each task, read the changed source files.
3. Update the wiki page with `wiki pages update`.
4. Re-run `ingest` — it should report 0 pages needing attention.

## Verifying Wiki Health

```bash
# Run all 6 structural checks
indexion wiki lint --wiki-dir=.indexion/wiki
```

The six checks and what they mean:

| Check | What it finds |
|-------|--------------|
| Broken links | `wiki://page-id` references to pages that don't exist |
| Orphan pages | Pages unreachable from nav tree and not linked by any other page |
| Missing cross-references | Pages sharing source files that don't link to each other |
| Stale sources | `sources` paths that no longer exist on disk |
| Empty pages | Pages with near-zero content |
| Manifest-file mismatch | `wiki.json` entries with no `.md` file (or vice versa) |

**After writing new pages, always run `wiki lint` and fix any issues before finishing.**

Cross-reference warnings (Missing cross-references) are common when related concepts
are split across pages. Fix them by adding a `## See Also` section with `wiki://page-id` links.

## Full Wiki Maintenance Cycle

This is the recommended sequence for an agent maintaining a wiki:

```bash
# 1. Detect stale pages
indexion wiki pages ingest --wiki-dir=.indexion/wiki

# 2. Read the index to understand the wiki structure
cat .indexion/wiki/index.md

# 3. Update each stale page (repeat for each task from step 1)
indexion wiki pages update --id=<page-id> --content=/tmp/updated.md \
  --sources="..." --provenance=synthesized --actor="agent:<name>" \
  --wiki-dir=.indexion/wiki

# 4. Verify structural integrity
indexion wiki lint --wiki-dir=.indexion/wiki

# 5. Regenerate the index to reflect all changes
indexion wiki index build --wiki-dir=.indexion/wiki

# 6. Confirm everything is current
indexion wiki pages ingest --dry-run --wiki-dir=.indexion/wiki
```

## Planning a New Wiki from Scratch

When the wiki doesn't exist yet or needs a structural overhaul:

```bash
# Analyze the project and generate a page structure proposal
indexion wiki pages plan --format=md <project-dir>

# Then execute the plan: add each proposed page
indexion wiki pages add --id=<id> --title="..." --content=/tmp/page.md \
  --sources="..." --provenance=synthesized --actor="agent:<name>"
```

`wiki pages plan` uses the CodeGraph to propose concept-based pages, not file-based ones.
Prefer concept pages (e.g., "KGF System") over file pages (e.g., "src/kgf/lexer/lexer.mbt").

## Building the Search Index

```bash
# Build navigation index only (index.md)
indexion wiki index build --wiki-dir=.indexion/wiki

# Build navigation + vector search index (vectors.db)
indexion wiki index build --full --wiki-dir=.indexion/wiki

# Preview index.md without writing
indexion wiki index build --dry-run --wiki-dir=.indexion/wiki
```

With `--full`, `indexion search .indexion/wiki/` will use the pre-built vectors
instead of rebuilding from scratch on each query.

## Checking the Audit Trail

```bash
# Recent operations
indexion wiki log --wiki-dir=.indexion/wiki --tail=20

# Full log as JSON
indexion wiki log --wiki-dir=.indexion/wiki --json
```

Every `pages add`, `pages update`, `lint`, `pages ingest`, and `index build` call appends to
`.indexion/wiki/log.json`. Use the log to verify that previous agent runs completed
correctly, or to understand why a page was last modified.

## Key Pitfalls

**Missing cross-references accumulate silently.** Any two pages that share source files
but don't link to each other will be flagged by `wiki lint`. After adding a page about
a topic that overlaps with existing pages, check for and add missing cross-references.

**Pages without `sources` drift invisibly.** `wiki pages ingest` can only detect change for
pages that list source files. A page with no `sources` will never appear in the ingest
task list, even if the underlying code has completely changed.

**Backtick-quoted `wiki://` references are not links.** The lint checker correctly
ignores ``wiki://`` in code spans. Use bare `wiki://page-id` or Markdown links
`[Title](wiki://page-id)` to create real cross-references.

**`wiki pages ingest` manifests state at the time of the last run.** If you add new source
files to a page's `sources` list via `pages update`, the next `ingest` run will show
that page as needing attention (because the new sources have no recorded hash). This is
expected behavior — run `ingest` again after the update to record the baseline.
