---
name: indexion-kgf
description: Inspect and debug KGF (Knowledge Graph Framework) specs — tokenize, parse, and extract edges from source files. Use when the user wants to debug language parsing, inspect how indexion processes a file, or verify KGF spec behavior.
---

# indexion kgf

Inspect and debug KGF language specs by viewing tokens, parse events, and extracted edges.

## When to Use

- User wants to debug how indexion processes a specific file
- User is developing or modifying a KGF spec
- User asks "how does indexion parse this file?"
- Verifying that tokenization/parsing works correctly for a language
- **Debugging grep patterns**: when a grep pattern doesn't match, use
  `kgf tokens` to see the actual token kinds

## Subcommands

### `indexion kgf inspect` — Full Inspection

Show tokens, events, and edges all at once.

```bash
indexion kgf inspect <file>
indexion kgf inspect --spec=typescript src/app.ts
```

### `indexion kgf tokens` — Tokenization Only

Show how a file is tokenized.

```bash
indexion kgf tokens <file>
indexion kgf tokens --spec=go-mod go.mod
```

### `indexion kgf events` — Parse Events Only

Show parse events generated from tokens.

```bash
indexion kgf events <file>
```

### `indexion kgf edges` — Extracted Edges Only

Show the dependency edges extracted from a file.

```bash
indexion kgf edges <file>
indexion kgf edges fixtures/project/npm/package.json
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--spec=NAME` | auto-detect | KGF spec name to use |
| `--kgf-dir=PATH` | kgfs | KGF specs directory |

## Relationship to grep

`indexion grep` uses KGF tokenization under the hood. Pattern aliases
(`pub` → `KW_pub`) are derived from the `=== lex` section of KGF specs.

When a grep pattern doesn't match as expected:

```bash
# 1. See the actual tokens for a file
indexion kgf tokens src/config/paths.mbt

# 2. Check which token kinds exist
indexion kgf tokens src/config/paths.mbt | head -20

# 3. Then adjust your grep pattern to match the actual token kinds
indexion grep "KW_pub KW_fn Ident" src/config/paths.mbt
```

Common token kinds (MoonBit):
- `KW_pub`, `KW_fn`, `KW_struct`, `KW_enum`, `KW_type`, `KW_trait`, `KW_let`, `KW_for`
- `Ident` (lowercase identifiers), `TypeIdent` (PascalCase type names)
- `LPAREN`, `RPAREN`, `LBRACE`, `RBRACE`, `LBRACKET`, `RBRACKET`
- `NL` (newline), `SKIP` (whitespace — filtered from grep patterns)
- `DocComment`, `DocLine`, `DocSection`, `LineComment`, `BlockComment`
- `String`, `Number`, `Char`

## Workflow

1. Run `indexion kgf inspect <file>` to see the full processing pipeline
2. If something looks wrong, drill down with `tokens`, `events`, or `edges`
3. Compare with the KGF spec file (`kgfs/<lang>.kgf`) to diagnose issues
