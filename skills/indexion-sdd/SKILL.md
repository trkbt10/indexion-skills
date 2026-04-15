---
name: indexion-sdd
description: Generate SDD requirements from RFCs/specs and quantitatively verify implementation conformance. spec draft → spec align → spec verify → automated validation loop with codex/claude. Operate spec-to-impl drift gates as CI checks.
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

## Important

- **Set cc-sdd `--lang` to match the implementation language, not the
  spec language.** If code is in English, use `--lang en`. Mismatched
  languages break `spec align diff` vocabulary matching.

## Process: RFC → Implementation → Validation

### Step 0: Project Setup

```bash
# Create project directory
mkdir my-rfc-impl && cd my-rfc-impl

# Install cc-sdd for your agent — set lang to match the code language
npx cc-sdd@latest --codex-skills --lang en --yes   # codex (skills mode)
npx cc-sdd@latest --claude-skills --lang en --yes   # claude (skills mode)

# Initialize MoonBit project (or other language)
# Create moon.mod.json, src/lib/moon.pkg.json, etc.

# Initialize git
git init && git add -A && git commit -m "init"
```

**Verify indexion is available** before proceeding. The drift gate
(Step 2.5) and validation loop (Step 3) depend on it:

```bash
indexion --version                       # must be installed
indexion kgf edges <any-source-file>     # must detect language and show edges
```

If `indexion` is not installed, install it first. If the KGF for your
spec format is not recognized, update `indexion` to a version that
includes it. See Step 2.7 for the fallback when this cannot be resolved.

### Step 1: Specification → SDD Draft (indexion)

Fetch the specification document (RFC, ISO standard, etc.) and prepare it:

```bash
# For RFC: save as markdown or .rfc.txt
indexion spec draft --output .kiro/specs/<feature>/requirements.md rfc_document.md

# For ISO/IEC standards: extract text from PDF with cleanup, then draft
python3 scripts/extract_iso_text.py spec.pdf <start_page> <end_page> spec.spec.txt
indexion spec draft --output .kiro/specs/<feature>/requirements.md spec.spec.txt

# Create spec.json (set requirements.generated=true, approved=true)
```

The output uses `### Requirement N:` and `#### N.M:` hierarchy matching
cc-sdd's expected ID format. The agent's `$kiro-spec-requirements` phase will
further refine into EARS format with acceptance criteria.

**Supported specification formats:**
| Format | Extension | KGF spec |
|--------|-----------|----------|
| RFC plaintext | `.rfc.txt` | `rfc-plaintext` |
| ISO/IEC technical document | `.spec.txt` | `technical-document` |
| Markdown (README, etc.) | `.md` | `markdown` |

### Step 1.5: Specification Fidelity Check (indexion)

After writing requirements, verify they cover the source specification:

```bash
# Check requirements ↔ original spec alignment
# (spec align now accepts document files as impl when no code files are found)
indexion spec align diff .kiro/specs/<feature>/requirements.md spec.spec.txt \
  --format markdown --threshold 0.3

# SPEC_ONLY items = requirements not covered by original spec
# DRIFTED items = requirements using different vocabulary from original spec
```

**Traceability chain:** `spec align` works between documents at the
same abstraction level. For full traceability, check each adjacent pair:

```bash
# 1. Source spec ↔ requirements (fidelity)
indexion spec align diff requirements.md source-spec.spec.txt --threshold 0.3
# 2. Requirements ↔ design (design coverage)
indexion spec align diff requirements.md design.md --threshold 0.3
# 3. Requirements ↔ implementation (impl coverage) — after $kiro-impl
indexion spec align diff requirements.md src/ --threshold 0.3
```

Do NOT align source spec directly against design or code — the vocabulary
spaces differ too much (normative spec language vs. software design vs.
code identifiers). Each hop bridges one abstraction gap.

### Step 2: Agent SDD Phases (codex / claude)

cc-sdd v3+ uses skills mode (`$kiro-spec-*` commands).

**Non-interactive execution with indexion gates between phases:**

```bash
FEATURE=<feature>
SPEC_DIR=.kiro/specs/$FEATURE
REPORT_DIR=.indexion/sdd-reports/$FEATURE
mkdir -p $REPORT_DIR

# Helper: approve a phase in spec.json
approve() { jq --arg p "$1" '.approvals[$p].approved = true' $SPEC_DIR/spec.json > /tmp/spec.json && mv /tmp/spec.json $SPEC_DIR/spec.json; }

# --- Design ---
codex exec --full-auto --json -C . "\$kiro-spec-design $FEATURE -y" > $REPORT_DIR/design.jsonl
approve design
# Verify requirements ↔ design
indexion spec align diff $SPEC_DIR/requirements.md $SPEC_DIR/design.md \
  --format markdown --threshold 0.3 | tee $REPORT_DIR/req-design-align.md
git add $SPEC_DIR && git commit -m "spec: design for $FEATURE"

# --- Tasks ---
codex exec --full-auto --json -C . "\$kiro-spec-tasks $FEATURE -y" > $REPORT_DIR/tasks.jsonl
git add $SPEC_DIR && git commit -m "spec: tasks for $FEATURE"

# --- Impl (long-running, background + monitor) ---
# Write a prompt file instead of using $kiro-impl directly.
# This allows including drift gate and commit instructions.
cat > $REPORT_DIR/impl-prompt.md << 'IMPLEOF'
## Context

You are implementing the <FEATURE> feature.
Read `.kiro/specs/<FEATURE>/tasks.md` for task details and
`.kiro/specs/<FEATURE>/design.md` for the design.

## Per-Task Drift Gate

After completing each task and committing, run:

```bash
indexion spec align diff .kiro/specs/<FEATURE>/requirements.md src/<pkg>/ --format markdown --threshold 0.3
indexion spec align status .kiro/specs/<FEATURE>/requirements.md src/<pkg>/ --threshold 0.3 --fail-on any
```

If DRIFTED or SPEC_ONLY items remain for the requirement your task
addresses, fix them (add spec vocabulary to pub declaration doc comments)
and re-commit before moving to the next task.

When all tasks are done, `spec align status --fail-on any` must exit 0.

## Important

Commit your work after completing tasks. Do not leave changes uncommitted.
IMPLEOF

# Replace <FEATURE> and <pkg> placeholders
sed -i '' "s/<FEATURE>/$FEATURE/g; s/<pkg>/$PKG/g" $REPORT_DIR/impl-prompt.md

codex exec --full-auto --json -C . \
  "$(cat $REPORT_DIR/impl-prompt.md)" > $REPORT_DIR/impl.jsonl &
CODEX_PID=$!
```

**Monitor impl progress:**

```bash
# Real-time event stream
tail -f $REPORT_DIR/impl.jsonl | jq -r '
  if .type == "item.completed" and .item.type == "agent_message" then
    "MSG: " + (.item.text[:120])
  elif .type == "item.completed" and .item.type == "command_execution" then
    "CMD: " + (.item.command[:60]) + " -> " + (.item.exit_code|tostring)
  else empty end'

# Stall detection
ps -p $CODEX_PID -o cputime,%cpu
lsof -p $CODEX_PID 2>/dev/null | grep tcp

# Resume after interruption
codex exec resume --last
```

**Operational notes:**

- Always use `--json` with file redirect + `tail -f`. Piping through
  `| tail` suppresses all output until completion.
- For long prompts, write to a file and use `$(cat file.md)`. Shell
  HEREDOC with `$()` can cause stdin blocking.
- Use `/loop` (Claude Code) to poll `git log --oneline -1` + `ps`
  every 60-120s during long impl runs.

### Step 2.5: Per-Task Drift Gate (indexion)

After each `$kiro-impl` task commit, run spec alignment to verify the
task closed its corresponding requirement gap. Do NOT proceed to the
next task if the requirement addressed by the current task is still
DRIFTED or SPEC_ONLY.

```bash
FEATURE=<feature>
SPEC_DIR=.kiro/specs/$FEATURE

# After each task commit:
indexion spec align diff $SPEC_DIR/requirements.md src/ --format markdown --threshold 0.3
indexion spec align status $SPEC_DIR/requirements.md src/ --threshold 0.3 --fail-on any
```

When including this gate in a Codex prompt, instruct the agent to run
these commands after each task commit and fix any DRIFTED items before
proceeding. Example instruction block for the prompt:

```
After completing each task (commit), run:
  indexion spec align diff ... --threshold 0.3
  indexion spec align status ... --threshold 0.3 --fail-on any
If DRIFTED or SPEC_ONLY items remain for the requirement your task
addresses, fix them (typically by adding spec vocabulary to pub
declaration doc comments) before moving to the next task.
When all tasks are done, spec align status --fail-on any must exit 0.
```

### Step 2.6: Stall Detection and Recovery

Codex processes can stall (lost API connection, blocked review loop, etc.).

**Detection:**

```bash
# Check process vitals
ps -p $CODEX_PID -o cputime,%cpu,etime
lsof -p $CODEX_PID 2>/dev/null | grep tcp

# Stall indicators (all must be true):
# - CPU time not increasing over 2+ checks
# - 0.0% CPU
# - No TCP connections
# - JSONL event count not increasing
```

**Recovery — never commit manually:**

When a stall is confirmed:

1. Kill the process: `kill $CODEX_PID`
2. Check what was completed: `git log`, `git status`, `git diff --cached`
3. If uncommitted work exists and tests pass, generate indexion reports:
   ```bash
   indexion spec align diff $SPEC_DIR/requirements.md src/ \
     --format markdown --threshold 0.3 -o $REPORT_DIR/align-current.md
   indexion spec verify --spec="$SPEC_DIR/requirements.md" src/ \
     --format md -o $REPORT_DIR/verify-current.md
   ```
4. Write a resume prompt file that:
   - States which tasks are committed and which have uncommitted work
   - Includes the indexion reports as current-state evidence
   - Instructs Codex to verify and commit uncommitted work, then continue
   - Requires the per-task drift gate (Step 2.5) for all remaining tasks
5. Restart: `codex exec --full-auto --json -C . "$(cat $REPORT_DIR/resume-prompt.md)" > $REPORT_DIR/impl-resume.jsonl &`

**Never commit implementation code manually.** All commits must come from
the Codex agent. If you commit manually, you bypass the SDD protocol
(RED→GREEN evidence, review, verification) and invalidate the workflow.

### Step 2.7: Drift Gate Proxy (when Codex cannot run indexion)

Codex runs `indexion` commands inside the impl project directory. If
`indexion` is not installed, is too old, or lacks required KGF specs,
the drift gate commands will fail or timeout inside Codex.

**Prevention (recommended):** Verify in Step 0 that `indexion` works in
the project directory before starting the SDD run:

```bash
indexion --version
indexion kgf edges <any-project-file>   # should show edges, not errors
```

If `indexion` is not available or broken, install or update it first.

**Fallback — orchestrator-side drift gate:**

If `indexion` cannot be fixed before the run (e.g., a required KGF spec
is not yet released), the orchestrator runs the drift gate externally:

1. Tell Codex to skip drift gate commands in the impl prompt:
   ```
   The `indexion` binary is not available in this environment.
   Skip drift gate commands. The orchestrator will run them externally.
   ```
2. After each Codex commit, the orchestrator runs:
   ```bash
   indexion spec align diff $SPEC_DIR/requirements.md src/ \
     --format markdown --threshold 0.3
   indexion spec align status $SPEC_DIR/requirements.md src/ \
     --threshold 0.3 --fail-on any
   ```
3. If DRIFTED/SPEC_ONLY items appear, run the vocab fix step (Step 3.5)
   with the externally generated reports.

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

### Step 3.5: Vocabulary Alignment Fix (indexion + agent)

After `$kiro-validate-impl` passes but `spec align` shows DRIFTED/SPEC_ONLY,
the implementation is functionally correct but lacks spec vocabulary in doc
comments. This step closes that gap.

**Note:** `$kiro-validate-impl` (cc-sdd) does not invoke indexion's spec align
internally. This step bridges the two tools.

```bash
# 1. Generate alignment reports
indexion spec align diff .kiro/specs/<feature>/requirements.md src/ \
  --format markdown --threshold 0.3 -o .indexion/sdd-reports/align-diff.md
indexion spec verify --spec='.kiro/specs/<feature>/requirements.md' src/ \
  --format md -o .indexion/sdd-reports/verify-gaps.md

# 2. Write a prompt file (avoid HEREDOC — use file to prevent stdin blocking)
cat > .indexion/sdd-reports/vocab-fix-prompt.md << 'EOF'
The spec alignment tool reports that several requirements are DRIFTED or
SPEC_ONLY. For each item, determine whether it is:

1. **Implementation gap** — the spec requirement is not implemented at all.
   Add the missing implementation (types, functions, validation) and tests.
2. **Vocabulary gap** — the implementation exists but spec vocabulary is
   missing from public doc comments. Add spec terms to pub declarations.

Your task:
- Read the alignment report below.
- For each DRIFTED or SPEC_ONLY requirement, search the codebase for
  related keywords from that requirement. If related code exists, it is
  a vocabulary gap — add doc comments. If no related code exists, it is
  an implementation gap — implement it.
- The alignment tool only extracts vocabulary from public declarations.
  Doc comments on private functions are invisible to the tool.
- Run the project's formatter and test suite after editing.
- Commit your changes. Do not leave changes uncommitted.

## Spec Align Diff Report
EOF
cat .indexion/sdd-reports/align-diff.md >> .indexion/sdd-reports/vocab-fix-prompt.md
echo -e "\n## Vocabulary Gap Report" >> .indexion/sdd-reports/vocab-fix-prompt.md
head -40 .indexion/sdd-reports/verify-gaps.md >> .indexion/sdd-reports/vocab-fix-prompt.md

# 3. Run agent with the prompt file
codex exec --full-auto --json -C . \
  "$(cat .indexion/sdd-reports/vocab-fix-prompt.md)" \
  > .indexion/sdd-reports/vocab-fix.jsonl &

# 4. Monitor (optional)
tail -f .indexion/sdd-reports/vocab-fix.jsonl | jq -r '
  if .type == "item.completed" and .item.type == "agent_message"
  then "MSG: " + (.item.text[:120])
  elif .type == "item.completed" and .item.type == "command_execution"
  then "CMD: " + (.item.command[:50]) + " -> " + (.item.exit_code|tostring)
  else empty end'

# 5. Re-check alignment after agent completes
indexion spec align status .kiro/specs/<feature>/requirements.md src/ \
  --threshold 0.3 --fail-on any
```

**Key constraints:**
- The prompt does NOT dictate specific vocabulary to add. It provides the
  alignment reports and lets the agent determine what to fix.
- The agent must target public declarations only — private function doc
  comments are invisible to `spec align`.
- Typical: 1-2 fix cycles. If SPEC_ONLY persists, the agent may need
  guidance that the relevant public API entry point should carry the
  vocabulary (e.g. a top-level `open` function documents the full flow).

### Step 4: Documentation Drift Detection (plan reconcile)

After implementation, use `plan reconcile` to detect drift between
implementation code and its documentation (README, doc comments, etc.):

```bash
# Detect impl ↔ documentation drift
indexion plan reconcile --format md src/lib/

# Restrict to specific docs
indexion plan reconcile --format md --doc 'README.md' src/lib/
```

`plan reconcile` complements `spec align` in the SDD workflow:
- **spec align**: requirements.md ↔ implementation code (spec conformance)
- **plan reconcile**: implementation code ↔ documentation (doc freshness)

Use `spec align` for pre-merge gates (CI). Use `plan reconcile` for
ongoing documentation maintenance after the feature ships.

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
| `-o, --output` | stdout | Output file path |

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

### spec align watch — Watch Mode

```bash
indexion spec align watch --mode=diff requirements.md src/lib/
indexion spec align watch --mode=status requirements.md src/lib/
```

Reruns alignment when spec or implementation inputs change.

### spec align suggest — Reconciliation Suggestions

```bash
indexion spec align suggest requirements.md src/lib/ --format tasks --agent codex
```

Generates actionable suggestions to close spec↔impl gaps.

## Threshold

Use `--threshold 0.3` for SDD alignment commands. The default `0.6` is
too high because SDD acceptance criteria are short text matched against
code + doc comments, not full-document similarity.

## Multi-Language Support

The SDD pipeline works with any language that has a KGF spec.

### KGF Requirements for SDD

For spec align to work, the language's KGF must:
1. **Capture doc comments in declares edges** — `doc` must be non-empty
2. **Use correct PEG alternative ordering** — DocComment must be after
   declaration rules in the Item alternatives
3. **Handle NL between doc and keyword** — `NL?` or `NL*` after `doc:DocComment?`

Verify with: `indexion kgf edges <file>` — if declares edges lack `doc=`,
the KGF needs fixing.
