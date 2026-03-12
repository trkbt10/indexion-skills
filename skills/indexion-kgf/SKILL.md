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

## Workflow

1. Run `indexion kgf inspect <file>` to see the full processing pipeline
2. If something looks wrong, drill down with `tokens`, `events`, or `edges`
3. Compare with the KGF spec file (`kgfs/<lang>.kgf`) to diagnose issues
