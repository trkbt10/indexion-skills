---
name: indexion-search
description: "この関数どこ？" "O(n^2)ありそうな箇所は？" "似てるファイルは？" "こういう処理してる関数は？" — コードベースを探す意図に応じて適切なindexionコマンドを選び、検索する。キャッシュインデックスのドリフト管理も含む。
---

# indexion search — Codebase Search & Exploration

Find what you need in a codebase. This skill covers every search intent —
pattern matching, structural queries, semantic similarity, file-level exploration,
and purpose-based function lookup — and tells you which tool to reach for based on
what you're actually trying to do.

## Start Here: What Are You Looking For?

Don't pick a command first. Start with your intent.

### "Where is X defined / used?"

You know the name. You need to find it.

**名前の完全一致** — `Ident:X` は完全一致。定義と全呼び出し箇所を探す：

```bash
# 型の定義と全参照を探す
indexion grep "TypeIdent:DigestManifest" src/

# 関数の定義のみ探す（fn キーワード + 識別子）
indexion grep "fn Ident:parse_config" cmd/

# 関数の定義も呼び出しも全部探す（fn なしで Ident だけ）
indexion grep "Ident:parse_config" cmd/

# pub struct の定義一覧
indexion grep "pub struct *" src/
```

**名前の部分一致** — `--semantic=name:X` は部分一致。正確な名前を覚えていないときに：

```bash
# "build_graph" を含む関数を全て探す（build_graph_from_source も見つかる）
indexion grep --semantic=name:build_graph src/ cmd/

# "sort" を含む関数を全て探す
indexion grep --semantic=name:sort src/
```

`Ident:build_graph` は `build_graph` という名前のトークンだけに完全一致する。
`build_graph_from_source` を見つけたいなら `--semantic=name:build_graph` を使う。

**token-level search** — `indexion grep` はテキストの grep ではなく、KGF スペックが
定義するトークン構造に基づいて検索する。`pub fn *` はキーワード `pub`、キーワード `fn`、
任意の識別子にマッチし、文字列リテラルやコメント内の `pub fn` にはマッチしない。

**Pattern syntax:**

| Pattern | Meaning |
|---------|---------|
| `pub`, `fn`, `for` | Language keyword (auto-resolved via KGF aliases) |
| `KW_fn` | Token kind exactly |
| `Ident:foo` | Kind + text (exact match) |
| `*` | Any single token |
| `...` | Zero or more tokens (non-greedy) |
| `!pub` | Negation — any token except this |
| `(`, `)`, `{`, `->` | Punctuation (aliases) |

**Output control:**

```bash
# File paths only (like grep -l)
indexion grep --files "pub fn *" src/

# Match count per file (like grep -c)
indexion grep --count "pub fn *" src/

# Context lines around matches
indexion grep --context=3 "for ... for" src/

# Filter by file pattern
indexion grep --include='*.mbt' --exclude='*_test.mbt' "pub fn *" src/
```

### "Find code that does X" (by description)

You don't know the name. You know what it does.

Two tools serve this intent, with a key trade-off:

| Tool | Build step? | Speed | Best for |
|------|------------|-------|----------|
| `grep --semantic="similar:..."` | No | Moderate | One-off queries, small codebases |
| `digest query` | Yes (cached) | Fast after build | Repeated queries, large codebases |

**One-off (no build step):**

```bash
# Find functions that parse JSON configuration
indexion grep --semantic="similar:parse JSON configuration" src/

# Find error handling patterns
indexion grep --semantic="similar:handle error and return result" src/

# Find tokenization-related code
indexion grep --semantic="similar:tokenize source code into tokens" src/
```

Results are ranked by cosine similarity. Use descriptive phrases —
"parse JSON configuration" works better than just "json".

**Repeated queries (cached index):**

```bash
# First: build the index (once, then incremental updates)
indexion digest build src/

# Then: query by purpose
indexion digest query "parse configuration file"
indexion digest query "calculate similarity between two texts"
indexion digest query "walk directory tree and collect files"
```

`digest query` auto-updates the index if source files have changed since the
last build — you don't need to manually rebuild. See [Index Lifecycle](#index-lifecycle)
for details.

**Full-text semantic search (code + wiki + docs):**

```bash
# Search across everything — code, wiki pages, documentation
indexion search "how does the KGF parser work" src/ .indexion/wiki/

# Filter by attributes
indexion search --filter="node_type:code" "tokenizer" src/
indexion search --filter="language:moonbit" "parse" src/

# File paths only
indexion search --files "configuration parsing" src/

# JSON output for scripting
indexion search --json "error handling" src/

# Control result count and minimum relevance
indexion search --top-k=50 --min-score=0.1 "similarity" src/
```

`search` differs from `digest query`: it searches across file content
(not just function purpose), and can include wiki pages and documentation.
`digest query` is function-granular and purpose-indexed.

### "Find structural patterns" (code smells, complexity)

You're looking for a *shape* of code, not specific text.

```bash
# Nested for loops (O(n^2) candidates)
indexion grep "for ... for" src/

# Undocumented public API
indexion grep --undocumented src/

# Functions longer than 50 lines
indexion grep --semantic=long:50 src/

# Functions shorter than 3 lines (trivial?)
indexion grep --semantic=short:3 src/

# Functions with 4+ parameters (complexity smell)
indexion grep --semantic=params-gte:4 src/

# Proxy functions (wrappers that just delegate)
indexion grep --semantic=proxy src/

# Functions by name substring
indexion grep --semantic=name:sort src/
indexion grep --semantic=name:parse src/
```

These are **semantic queries** — they analyze function structure, not just text.
`--semantic=proxy` examines whether a function body is a single forwarded call.
`--semantic=long:50` counts lines of the function body.

**Combining for code review:**

```bash
# Post-commit quality check
indexion grep "for ... for" src/                    # O(n^2) hotspots
indexion grep --undocumented src/                    # Missing docs
indexion grep --semantic=proxy src/                  # Unnecessary wrappers
indexion grep --semantic=long:50 src/                # Overly long functions
indexion grep --semantic=params-gte:4 src/           # Complex signatures
```

### "Which files are similar to each other?"

You want to understand overlap and relationships across files.

```bash
# Quick similarity scan — sorted pairs above threshold
indexion explore --format=list --threshold=0.7 \
  --include='*.mbt' --exclude='*moon.pkg*' src/

# Cluster similar files together
indexion explore --format=cluster --threshold=0.6 src/

# Full similarity matrix (small file sets only)
indexion explore --format=matrix src/module_a/ src/module_b/

# JSON for scripting
indexion explore --format=json --threshold=0.5 src/

# Compare specific files
indexion explore file_a.mbt file_b.mbt --threshold=0

# Function-level structural comparison (slower, more precise)
indexion explore --strategy=apted --format=list src/
indexion explore --strategy=tsed --format=list src/

# Statistical correction for large codebases
indexion explore --fdr=0.05 --format=list --threshold=0.5 src/
```

**Strategies — when the default isn't enough:**

| Strategy | What it compares | When to use |
|----------|-----------------|-------------|
| `hybrid` (default) | BM25 + JSD lexical, TSED structural fusion | General purpose — start here |
| `tfidf` | Token vocabulary overlap | Fast scan, vocabulary-level |
| `bm25` | Term frequency relevance | Similar to tfidf, different weighting |
| `jsd` | Token distribution divergence | Probabilistic similarity |
| `ncd` | Compression-based similarity | Language-agnostic, byte-level |
| `apted` | Function-level tree edit distance | Precise structural comparison |
| `tsed` | Tree structure edit distance | Precise structural comparison |

**Noise reduction tips:**
- `--exclude='*moon.pkg*'` — package config files all look alike
- `--exclude='*_wbtest.mbt'` — test files share boilerplate
- Files at 40-60% with no structural overlap are "concept neighbors" — they share
  vocabulary because they work in the same domain (see indexion-refactor skill)

### "Compare two specific texts"

Direct pairwise similarity between two texts or files.

```bash
# Compare two texts
indexion sim "hello world" "hello there"

# Compare file contents (shell reads)
indexion sim "$(cat a.mbt)" "$(cat b.mbt)"

# Choose strategy
indexion sim --strategy=ncd "text1" "text2"
indexion sim --strategy=tfidf "text1" "text2"

# Output control
indexion sim --output=similarity "text1" "text2"   # 0.0-1.0 only
indexion sim --output=distance "text1" "text2"     # distance only
indexion sim --output=both "text1" "text2"         # both (default)
```

### "Trace references before refactoring"

Before moving or renaming something, find all references.

```bash
# 型の全参照を探す（定義 + 使用箇所）
indexion grep "TypeIdent:DigestManifest" src/ cmd/

# 関数の全参照を探す（定義 + 呼び出し箇所）— Ident:X は完全一致
indexion grep "Ident:parse_config" cmd/

# 名前の一部しかわからない場合は --semantic=name: で部分一致
indexion grep --semantic=name:build_graph src/ cmd/

# リネーム後に旧名称の参照が残っていないか確認
indexion grep "Ident:old_function_name" src/ cmd/
```

**注意:** `Ident:X` と `TypeIdent:X` は異なるトークン種別。型名は `TypeIdent`、
変数・関数名は `Ident`。どちらかわからないときは `--semantic=name:X` を使う。

## Index Lifecycle

`digest` and `search` build cached indexes for fast repeated queries.
These indexes can become stale when source files change.

### How Drift Detection Works

The digest index maintains a **manifest** at `.indexion/digest/` that tracks:

| What | How | Detects |
|------|-----|---------|
| File content | SHA-256 hash per file | Modified, new, deleted files |
| Function identity | Hash of file_hash + symbol_id + name + kind | Changed function bodies |
| Call graph | Hash of source_hash + sorted callers + callees | Changed relationships |
| KGF spec | Spec fingerprint | Language grammar changes |

When you run `digest query`, it automatically compares current file hashes against
the manifest. If files have changed, it **incrementally re-indexes** only the
affected functions — no full rebuild needed.

### Check Index Health

```bash
# Show what has changed since the last index build
indexion digest status src/

# Show index statistics (size, function count, providers)
indexion digest stats
```

`digest status` reports:
- Modified files (content hash differs)
- New files (not in manifest)
- Deleted files (in manifest but gone from disk)
- Estimated affected functions

### When to Rebuild vs. Incremental Update

| Situation | Action | Command |
|-----------|--------|---------|
| Normal development | Auto-update on query | `indexion digest query "..."` (auto) |
| Skip auto-update for speed | Query saved index only | `indexion digest query --no-update "..."` |
| Many files changed | Manual incremental build | `indexion digest build src/` |
| KGF spec changed | Full rebuild | `indexion digest rebuild src/` |
| Index corrupted or version mismatch | Full rebuild | `indexion digest rebuild src/` |
| First time setup | Initial build | `indexion digest build src/` |

### Keep Index Fresh During Development

After significant code changes (new module, major refactor), run:

```bash
# Check what's stale
indexion digest status src/

# If many changes: explicit build (faster than auto-update during query)
indexion digest build src/

# Verify
indexion digest stats
```

For wiki search indexes, rebuild with:

```bash
indexion wiki index build --full --wiki-dir=.indexion/wiki
```

## Choosing the Right Tool

| Intent | Tool | Why |
|--------|------|-----|
| Find by name | `grep "Ident:X"` | Token-level, precise |
| Find by keyword pattern | `grep "pub fn *"` | Token pattern matching |
| Find by description (one-off) | `grep --semantic="similar:..."` | No build step |
| Find by description (repeated) | `digest query` | Cached, fast after build |
| Find across code + docs + wiki | `search` | Broadest scope |
| Find structural patterns | `grep --semantic=long:50` | Structural analysis |
| Find similar files | `explore --format=list` | Pairwise similarity |
| Compare two specific texts | `sim` | Direct comparison |
| Trace all references | `grep "TypeIdent:X"` | Before refactoring |

## Common Pitfalls

**"grep found nothing but I know it exists"**
- **トークン種別の不一致:** `Ident:Config` は `TypeIdent:Config` にマッチしない。
  種別がわからなければ `--semantic=name:Config` を使う（種別を問わず部分一致）。
- **完全一致 vs 部分一致:** `Ident:build_graph` は `build_graph_from_source` にマッチしない。
  部分一致したいなら `--semantic=name:build_graph` を使う。
- **対象ディレクトリの間違い:** `src/` に無いものを `src/` で探していないか確認。
  `src/ cmd/` のように複数ディレクトリを指定できる。

**"explore shows 95% similarity between types.mbt files"**
- Type definition files share structural patterns (pub struct + getters).
  This inflates TF-IDF scores. It's structural similarity, not duplication.
  Exclude with `--exclude='*types.mbt'` or use `--strategy=apted` for function-level.

**"digest query returns irrelevant results"**
- Use descriptive purpose phrases, not code keywords.
  "calculate text similarity using TF-IDF" beats "tfidf".
- Check `digest status` — the index may be stale.

**"search is slow on a large codebase"**
- Build a digest index first: `indexion digest build src/`
- Use `--include` to narrow scope: `indexion search --include='*.mbt' "query" src/`

**"`...` in grep matches too much"**
- `...` is non-greedy — it matches the *shortest* span. `for ... for` finds the
  closest pair of for loops, not the furthest. This is usually correct for nesting detection.
