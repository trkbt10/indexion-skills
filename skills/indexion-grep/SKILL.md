---
name: indexion-grep
description: KGF-aware token pattern and semantic code search. Use when the user wants to search code by structure (not just text), find specific patterns like nested loops or undocumented functions, or search by natural language similarity.
---

# indexion grep

KGF-aware token pattern search, structural queries, and vector similarity search.

## When to Use

- User asks to find specific code patterns (e.g. "nested for loops", "pub fn without docs")
- User asks "where is this function used?" or "find all pub structs"
- User wants structural search — not regex on raw text, but token-level matching
- User asks for semantic code search ("find functions that parse configuration")
- User wants to find proxy functions, long functions, or functions with many params
- **Replaces manual grep/ripgrep** for code-aware searches
- **Use instead of Explore agent** for targeted codebase queries

## Token Pattern Search

Patterns are space-separated token matchers. KGF aliases resolve automatically
(e.g. `pub` → `KW_pub`), so you write natural code keywords:

```bash
# Find all pub fn declarations
indexion grep "pub fn *" src/

# Find pub struct definitions
indexion grep "pub struct *" src/

# Nested for loops (O(n²) candidates)
indexion grep "for ... for" src/

# Functions named "sort"
indexion grep "fn Ident:sort" src/

# Any token except pub, followed by fn
indexion grep "!pub fn" src/

# Using raw KGF token kinds also works
indexion grep "KW_pub KW_fn Ident" src/
```

### Pattern Syntax

| Pattern | Meaning |
|---------|---------|
| `pub` | Match keyword (auto-alias → `KW_pub`) |
| `KW_fn` | Match token kind exactly |
| `Ident:foo` | Match kind and text |
| `*` | Match any single token |
| `...` | Match zero or more tokens (non-greedy) |
| `!pub` | Negation — any token except this kind |
| `(`, `)`, `{`, `->` | Punctuation aliases |

Aliases are auto-generated from KGF specs — not hardcoded. Works for all
KGF-supported languages.

## Semantic Queries

Structural analysis beyond token patterns:

```bash
# Find proxy functions (wrappers that just delegate)
indexion grep --semantic=proxy src/

# Find long functions (30+ lines)
indexion grep --semantic=long:30 src/

# Find short functions (3 lines or less)
indexion grep --semantic=short:3 src/

# Find functions with 4+ parameters
indexion grep --semantic=params-gte:4 src/

# Find functions by name substring
indexion grep --semantic=name:sort src/

# Find undocumented pub declarations (also available as --undocumented)
indexion grep --undocumented src/
indexion grep --semantic=undocumented src/
```

## Vector Similarity Search

Find code by natural language description using TF-IDF embeddings
(shared infrastructure with `digest`):

```bash
# Find functions related to "parse JSON configuration"
indexion grep --semantic="similar:parse JSON configuration" src/

# Find tokenization-related code
indexion grep --semantic="similar:tokenize source code into tokens" src/

# Find error handling patterns
indexion grep --semantic="similar:handle error and return" src/
```

Results are ranked by cosine similarity score.

## Output Control

```bash
# File paths only
indexion grep --files "pub fn *" src/

# Match count per file
indexion grep --count "pub fn *" src/

# Context lines around matches
indexion grep --context=3 "for ... for" src/

# Include/exclude patterns
indexion grep --include='*.mbt' --exclude='*_test.mbt' "pub fn *" src/
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--semantic=QUERY` | — | Semantic query (see above) |
| `--undocumented` | false | Find pub declarations without doc comments |
| `--files` | false | Show matching file paths only |
| `--count` | false | Show match count per file only |
| `--context=N` | 0 | Lines of context around matches |
| `--include=GLOB` | — | Include file pattern (repeatable) |
| `--exclude=GLOB` | — | Exclude file pattern (repeatable) |
| `--specs-dir=DIR` | kgfs | KGF specs directory |

## Relationship to Other Commands

| Command | Purpose | When to use |
|---------|---------|-------------|
| `grep` | Find specific patterns/functions | "Find all nested for loops" |
| `explore` | File-level similarity matrix | "What files are similar?" |
| `plan refactor` | Actionable refactoring plan | "What duplicates should I fix?" |
| `plan unwrap` | Proxy function detection + auto-fix | "Remove unnecessary wrappers" |
| `digest query` | Purpose-based function index | "Find function that handles X" (requires build step) |

`grep --semantic=proxy` overlaps with `plan unwrap` in detection, but `plan unwrap`
adds auto-fix capability. Use `grep` for quick discovery, `plan unwrap` for action.

`grep --semantic=similar:...` overlaps with `digest query` but requires no build step.
`digest` is better for repeated queries on large codebases (cached index).

## Dogfooding Workflow

```bash
# After writing new code, check for patterns that need attention:

# 1. Find potential O(n²) sorts
indexion grep "for ... for" src/

# 2. Check for undocumented public API
indexion grep --undocumented src/

# 3. Find proxy functions to consider unwrapping
indexion grep --semantic=proxy src/

# 4. Find overly long functions
indexion grep --semantic=long:50 src/

# 5. Search for specific refactoring targets
indexion grep --semantic="similar:extract substring" src/
```
