---
name: indexion-sdd
description: Spec-Driven Development with indexion quantitative analysis. Use when the user wants to run SDD workflows, verify spec-to-implementation conformance, generate SDD drafts from documents, or integrate indexion with cc-sdd/codex for automated validation loops.
---

# indexion SDD (Spec-Driven Development)

Bridge indexion's quantitative spec alignment tools with SDD agent workflows (cc-sdd, codex, etc.).

Provides: RFC/document → SDD draft, spec↔impl drift detection, and a validation loop
that feeds indexion's quantitative results into agent-driven qualitative review.

## When to Use

- User wants to implement an RFC or specification as a MoonBit (or other) package
- User asks to verify spec-to-implementation conformance
- User wants to set up a SDD project with cc-sdd + codex + indexion
- User says "run the SDD loop" or "validate against the spec"
- After `codex spec-impl` completes, to check for drift before merging

## 注意事項

- **cc-sdd の `--lang` は実装コードの言語に合わせること。**
  英語コードなら `--lang en`。仕様が日本語でコードが英語だと
  `spec align diff` の語彙マッチが機能しない。

## Process: RFC → Implementation → Validation

### Step 0: Project Setup

```bash
# Create project directory
mkdir my-rfc-impl && cd my-rfc-impl

# Install cc-sdd for your agent — lang はコードの言語に合わせる
npx cc-sdd@latest --codex --lang en --yes   # codex
npx cc-sdd@latest --claude --lang en --yes  # or claude

# Initialize MoonBit project (or other language)
# Create moon.mod.json, src/lib/moon.pkg.json, etc.

# Initialize git
git init && git add -A && git commit -m "init"
```

### Step 1: RFC → SDD Draft (indexion)

Fetch the RFC and save as a local markdown file, then:

```bash
# Generate SDD requirements draft from the RFC document
indexion spec draft --output .kiro/specs/<feature>/requirements.md rfc_document.md

# Create spec.json (set requirements.generated=true, approved=true)
```

The output uses `### Requirement N:` and `#### N.M:` hierarchy matching
cc-sdd's expected ID format. The agent's `spec-requirements` phase will
further refine into EARS format with acceptance criteria.

### Step 2: Agent SDD Phases (codex / claude)

```bash
# Convert draft into EARS-format requirements
codex exec --full-auto "$(sed 's/\$1/<feature>/g' .codex/prompts/kiro-spec-requirements.md)"

# Generate design
codex exec --full-auto "$(sed 's/\$1/<feature>/g; s/\$2/-y/g' .codex/prompts/kiro-spec-design.md)"

# Generate tasks
codex exec --full-auto "$(sed 's/\$1/<feature>/g; s/\$2/-y/g; s/\$3//g' .codex/prompts/kiro-spec-tasks.md)"

# Approve tasks in spec.json (set tasks.approved=true, ready_for_implementation=true)

# Implement
codex exec --full-auto "$(sed 's/\$1/<feature>/g; s/\$2//g' .codex/prompts/kiro-spec-impl.md)"
```

### Step 3: Validation Loop (indexion + agent)

This is where indexion adds value beyond pure agent review.

```bash
# Quantitative analysis only (no agent)
indexion spec verify --spec='.kiro/specs/<feature>/requirements.md' src/lib/ --format md
indexion spec align diff .kiro/specs/<feature>/requirements.md src/lib/ --format markdown --threshold 0.3
indexion spec align status .kiro/specs/<feature>/requirements.md src/lib/ --threshold 0.3 --fail-on any

# Full loop: indexion quantitative + agent qualitative + auto-fix
./scripts/sdd-validate.sh <feature>           # validate only
./scripts/sdd-validate.sh <feature> --fix     # validate + auto-fix NO-GO tasks
```

The validation script:
1. Runs `spec verify` (vocabulary gap between spec and impl)
2. Runs `spec align diff` (requirement-level drift)
3. Runs `spec align trace` (traceability matrix)
4. Runs `spec align status` (CI-style pass/fail)
5. Injects all reports into the agent's `validate-impl` prompt
6. Agent produces GO/NO-GO decision with specific task numbers
7. On `--fix`, automatically re-runs `spec-impl` for failing tasks

## Individual Commands

### spec draft — RFC/Document → SDD Requirements Draft

```bash
indexion spec draft <source-file-or-dir>
indexion spec draft --output requirements.md --format markdown rfc.md
indexion spec draft --profile sdd-numbered-requirement rfc.md
```

| Option | Default | Description |
|--------|---------|-------------|
| `--output, -o` | stdout | Output file path |
| `--format` | markdown | Output format: markdown, json |
| `--profile` | sdd-numbered-requirement | Draft profile (KGF spec name) |
| `--max-requirements` | 64 | Maximum requirements to extract |
| `--specs-dir` | auto | KGF specs directory |

### spec verify — Vocabulary Gap Check

```bash
indexion spec verify --spec='requirements.md' src/lib/ --format md
```

| Option | Default | Description |
|--------|---------|-------------|
| `--spec` | (required) | Spec document glob (repeatable) |
| `--format` | json | Output format: json, md, github-issue |
| `--focus` | all | Token kind filter: ident, text, vocab, all |
| `--max-candidates` | 200 | Maximum items in output |

### spec align diff — Requirement-Level Drift

```bash
indexion spec align diff requirements.md src/lib/ --format markdown --threshold 0.3
```

Reports: MATCHED, DRIFTED, SPEC_ONLY (spec with no impl match), IMPL_ONLY (impl with no spec match).

### spec align trace — Traceability Matrix

```bash
indexion spec align trace requirements.md src/lib/ --format json --threshold 0.3
```

Generates requirement → implementation mapping for auditing.

### spec align status — CI Gate

```bash
indexion spec align status requirements.md src/lib/ --fail-on any --threshold 0.3
# Exit code non-zero on failure when used with --fail-on
```

| `--fail-on` | Description |
|-------------|-------------|
| `none` | Always pass |
| `drifted` | Fail if any DRIFTED |
| `spec-only` | Fail if any SPEC_ONLY |
| `any` | Fail on DRIFTED or SPEC_ONLY |

### spec align suggest — Reconciliation Suggestions

```bash
indexion spec align suggest requirements.md src/lib/ --format tasks --agent codex
```

Generates actionable suggestions to close spec↔impl gaps.

## sdd-validate.sh — The Validation Script

Place in `scripts/sdd-validate.sh` at the project root. Requires:
- `INDEXION_DIR` env var pointing to the indexion repo (or `indexion` in PATH)
- `codex` CLI in PATH
- cc-sdd prompts in `.codex/prompts/`

```bash
# Validate only
INDEXION_DIR=~/path/to/indexion ./scripts/sdd-validate.sh <feature>

# Validate + auto-fix
INDEXION_DIR=~/path/to/indexion ./scripts/sdd-validate.sh <feature> --fix
```

Reports are saved to `.indexion/sdd-reports/<feature>/`.

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `INDEXION_DIR` | (none) | Path to indexion repo (or `indexion` in PATH) |
| `SDD_THRESHOLD` | `0.3` | Similarity threshold for spec align |
| `SDD_IMPL_DIR` | auto-detect | Implementation directory (falls back to design.md → `src/lib`) |

## sdd-pipeline.sh — Full Autonomous Pipeline

Runs all 5 phases (requirements → design → tasks → impl → validate)
via codex, with indexion gates between each step.

```bash
# Full pipeline from scratch
INDEXION_DIR=~/path/to/indexion ./scripts/sdd-pipeline.sh <feature>

# Resume from a specific phase
INDEXION_DIR=~/path/to/indexion ./scripts/sdd-pipeline.sh <feature> impl

# Validate only (re-run after auto-fix)
INDEXION_DIR=~/path/to/indexion ./scripts/sdd-pipeline.sh <feature> validate
```

### Auto-Fix with Spec Alignment Context

When the validate phase produces NO-GO, the auto-fix re-runs `spec-impl`
with augmented context that includes the spec alignment reports. This is
critical: without the alignment context, codex sees "implementation is
correct" and makes no changes, leaving the alignment gate stuck at fail.

The augmented auto-fix prompt includes:
- The SPEC_ONLY items from `spec align diff`
- The vocabulary gaps from `spec verify`
- The alignment status (`status=fail`)
- Explicit instruction to add spec vocabulary to doc comments and test names

Without this augmentation, the auto-fix loop cannot close vocabulary gaps.

## Multi-Language Support

The SDD pipeline works with any language that has a KGF spec. Verified languages:

| Language | Doc Token | Notes |
|----------|-----------|-------|
| MoonBit | `///\|` + `///` | DocBlock order matters: `///\|` first |
| TypeScript | `/** */` | ExportDecl requires bind/scope for doc propagation |
| Python | `"""docstring"""` | Suite docstring requires bind/scope propagation |
| JavaScript (JSX) | `/** */` | Same ExportDecl pattern as TypeScript |
| Go, Rust, Swift, etc. | Various | PEG Item ordering fixed for all languages |

### Language-Specific KGF Requirements for SDD

For spec align to work, the KGF must:
1. **Capture doc comments in declares edges** — `doc` must be non-empty
2. **Use correct PEG alternative ordering** — DocComment must be after
   declaration rules in the Item alternatives
3. **Handle NL between doc and keyword** — `NL?` or `NL*` after `doc:DocComment?`

If `kgf edges <file>` shows declares without `doc=`, the KGF needs fixing.

## Dogfooding Lessons

Findings from 7 RFC implementations across 3 languages:

### Scoring & Matching (spec align)

- **BM25 replaces TF-IDF**: BM25's document length normalization gives
  stable scores when comparing short criteria against long code declarations.
- **Per-criterion boost**: Each acceptance criterion is scored individually
  against each implementation via BM25. Best criterion score lifts the
  requirement if it exceeds the whole-text score.
- **Identifier direct match**: Backtick-enclosed identifiers from criteria
  are extracted and matched against implementation names and body + doc.
  Unicode-safe `extract_alnum` strips punctuation while preserving CJK/etc.
- **Kneedle disabled for spec verify**: All gap candidates are returned.
  Spec compliance checking treats every gap as signal, not noise.
- **Strict kind matching for gaps**: `Ident:status` in spec must match
  `Ident:status` in impl (not `TEXT:status`).

### KGF Fixes (Cross-Language)

- **PEG Item ordering (13 languages)**: DocComment must appear AFTER
  declaration rules in the `Item` alternatives. In PEG, first-match-wins —
  a standalone DocComment alternative before FuncDecl will consume doc
  comments that should be attached to the following declaration.
  Fixed: Go, Haskell, Lua, OCaml, Rust, Scala, Zig, TypeScript,
  TypeScript-JSX, JavaScript-JSX, C#, Dart, Java, Kotlin, PHP, Swift, C, C++.
- **NL between doc and keyword**: Declaration rules need `NL?` or `NL*`
  between `doc:DocComment?` and the keyword (e.g., `KW_fn`). Without it,
  a newline between the doc comment and `export` prevents matching.
- **ExportDecl doc propagation (TypeScript/JS family)**: ExportDecl
  consumes the DocComment, but child FunctionDecl emits the declares edge.
  Fix: child rules bind id/kind/doc to scope, ExportDecl re-emits the
  declares edge with its own doc when `$doc` is present.
- **Python Decorator args**: `@dataclass(frozen=True)` has arguments after
  the decorator name. Added `DecoratorLine` rule to consume `(args)`.
- **Python Suite docstring propagation**: Docstrings are inside the Suite
  body. Suite/ClassSuite bind doc to scope, FunctionDef/ClassDef retrieve
  it via `$scope("value", "suite_doc")`.
- **OCaml/Haskell EQ/Operator token conflict**: Generic `Operator` regex
  includes `=`, consuming it before `EQ` token. Fix: move specific
  punctuation tokens (EQ, PIPE, COLON) before generic Operator.
- **Haskell FunctionDecl equation form**: Second alternative
  (`func_id:VarIdent Pattern* EQ Expr`) lacked `doc:DocComment? NL?`.
- **MoonBit DocBlock `///|` first ordering**: MoonBit convention places
  `///|` (block marker) before `///` (content), but DocBlock grammar only
  accepted `///` before `///|`. Added `DocComment NL ( DocLine NL )*` as
  first alternative.
- **Longest declaration span**: `best_declaration_span` now prefers the
  longest span (signature + body), not shortest (body only).
- **Numbered criteria in SDD KGF**: Added `"1."`-`"9."` prefixes to
  `collectLinesAfterPrefixes` so cc-sdd's numbered acceptance criteria
  are extracted.

### Process & Tooling

- **Spec vocabulary is mandatory, not cosmetic**: indexion `status=fail`
  forces NO-GO. codex must add spec vocabulary to code (doc comments,
  test names, function names) to make the alignment tool pass. This
  creates the traceable link between spec and code that the alignment
  tool measures.
- **Auto-fix needs alignment context**: The original auto-fix used plain
  `spec-impl` prompts. codex saw "implementation correct, no changes needed"
  and the loop never converged. Fix: inject spec align diff, verify, and
  status reports into the auto-fix prompt with explicit instructions to
  add spec vocabulary to doc comments.
- **NO-GO → fix → validate loop works**: With the augmented auto-fix
  prompt, codex adds doc comments with spec vocabulary, and the alignment
  gate passes on retry. Typical: 1-2 fix cycles after initial impl.
- **Don't work around broken tools**: Fix the tool, not the workflow.
  When `kgf edges` doesn't show doc, fix the KGF — don't manually write
  doc comments to compensate.
- **Graph cache for search**: File-hash-based cache for CodeGraph
  avoids rebuilding on unchanged files.
- **`spec draft` profile**: Uses KGF spec semantics for criteria
  extraction. `N.M` hierarchical IDs match cc-sdd format.

### Validated RFCs

| RFC | Language | Tests | Cycles to GO | Key Issue Found |
|-----|----------|-------|-------------|-----------------|
| 6901 JSON Pointer | MoonBit | 15 | 1 | search precision, spec draft dead code |
| 7396 JSON Merge Patch | MoonBit | 9 | 2 | failure boundary not implemented |
| 7807 Problem Details | MoonBit | 15 | 3 | public API boundary exposure |
| 2397 data: URL | MoonBit | 16 | 3 | DocBlock order, scope constraints traceability |
| 8288 Web Linking | TypeScript | 10 | 1 | PEG Item ordering, ExportDecl doc propagation |
| 6838 Media Type | Python | 22 | 1 | Decorator args, Suite docstring propagation |
| 3986 URI | MoonBit | 11 | 2 | DocBlock `///\|` first, auto-fix prompt gap |
