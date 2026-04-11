---
name: indexion-refactor
description: Detect domain-level concept duplication and enforce SoT (Single Source of Truth) across a codebase. Use when adding new features, introducing new abstractions, or after any change that touches multiple modules. Goes beyond textual similarity to find scattered ownership of the same concept.
---

# indexion refactor — SoT Enforcement

Detect and eliminate concept-level duplication using indexion's similarity analysis, then verify SoT is structurally enforced.

## When to Use

- After adding a new abstraction (type, module, API layer)
- After introducing a new file format or I/O boundary
- When a fix required touching 3+ files for the same reason
- When a "guard" or "skip" was added to work around a structural problem
- When `opendir`, `ENOENT`, or similar filesystem errors appear from unexpected paths
- Periodic SoT health check on a codebase

## The Problem This Solves

Textual duplication (`fn foo` copied between files) is easy to find. **Concept duplication** is harder:

- Two modules both decide "is this file an archive?"
- Three functions each contain "skip binary, collect text"
- A path string flows through 5 modules — each adds its own guard for invalid paths
- A `content` field exists in memory but 3 callers re-read it from disk

These produce no copy-paste matches but violate SoT. When the concept changes, every scattered copy must be updated — and the ones you miss become bugs.

## Workflow

### Phase 1: Detect concept domains with explore

```bash
# Find which files share vocabulary (= work in the same concept domain)
indexion explore --threshold=0.4 \
  --include='*.mbt' --exclude='*_wbtest.mbt' \
  --exclude='*moon.pkg*' --exclude='*pkg.generated*' \
  src/ cmd/
```

Files at 40-60% similarity without structural duplication are **concept neighbors** — they use the same terms because they deal with the same domain. List these pairs.

### Phase 2: Identify concept ownership

For each high-similarity pair, ask: **"What concept do they share, and who owns it?"**

```bash
# Inspect what vocabulary they share
indexion explore file_a.mbt file_b.mbt --threshold=0 --strategy=apted
```

Common patterns of concept leakage:

| Symptom | Concept leaked | Fix |
|---------|---------------|-----|
| Both files call `is_X(spec)` then `Y::from_spec(spec)` | "Determine if X and configure Y" | Extract `try_do_X(path, spec)` into the module that owns X |
| Both files `@fs.read_file_to_string(path)` when content is already loaded | "Read file content" | Pass content as argument, don't re-read |
| Both files `parent_dir(path)` then `@fs.read_dir(dir)` | "List sibling files" | Centralize directory walking into pipeline |
| Multiple `if is_virtual_path(x) { skip }` guards | "Real vs virtual path" | Make the type system prevent virtual paths from reaching here |
| Both files `buf.write_string("\n"); buf.write_string(x)` | "Join text entries" | Extract `join_text_entries()` into the owning module |

### Phase 3: Consolidate into SoT

The module that **defines the concept** should be the only one that **implements the logic**. Other modules call its API.

Rules:
1. **One concept, one module, one function.** If "extract text from archive" appears in `vfs.mbt`, `discover.mbt`, and `args.mbt`, it belongs in `vfs.mbt` only.
2. **Callers receive results, not ingredients.** Don't export `is_archive_spec` + `ArchiveSpec::from_spec` + `expand_archive` + "skip binary" separately. Export `try_extract_archive_text(path, spec) -> String?`.
3. **Guards are symptoms, not fixes.** `if is_virtual_path(x) { skip }` means virtual paths shouldn't reach here at all. Fix the source, not the sink.
4. **Re-reading from disk what's already in memory is a concept leak.** If `SupportedFile.content` holds the text, no downstream code should call `@fs.read_file_to_string(file.path)`.

### Phase 4: Verify with plan refactor

```bash
# Confirm textual duplication is gone
indexion plan refactor --threshold=0.9 \
  --include='*.mbt' --exclude='*_wbtest.mbt' \
  --exclude='*moon.pkg*' --exclude='*pkg.generated*' \
  src/ cmd/indexion/

# Confirm concept similarity is reduced
indexion explore file_a.mbt file_b.mbt --threshold=0
```

After SoT consolidation:
- Textual similarity between the concept owner and its callers drops
- Callers become shorter (one API call instead of multi-step logic)
- The concept owner may grow, but it's the **single place to change**

### Phase 5: Prove non-recurrence with tests

Write a test that **structurally prevents** the old pattern from recurring:

```moonbit
test "SoT: SupportedFile.path is always a real filesystem path" {
  // Create an archive, run load_supported_file_info
  // Assert: no path contains "!/"
  // Assert: every path passes @fs.path_exists
}
```

The test doesn't check behavior — it checks the **SoT invariant**. If someone later re-introduces virtual paths into `SupportedFile`, this test fails before the bug manifests downstream.

## Red Flags to Watch For

### "I need to add a guard here"

If you're adding `if is_special_case(x) { skip }` to a function that shouldn't receive special cases, **the problem is upstream**. The function's caller should never pass that value.

### "It works but prints errors to stderr"

Stderr messages from C runtime (`opendir: No such file or directory`, `stat: ...`) mean invalid data reached a system call. The `catch` block in MoonBit absorbs the error, but the C `perror()` already printed. **You cannot suppress this with catch.** The only fix is preventing invalid data from reaching the call.

### "I'll fix it in each command separately"

If the same fix is needed in explore, search, grep, reconcile, plan documentation... the fix belongs in the shared pipeline, not in each command. Each per-command fix is a ticking time bomb for the next command that's written.

### "The similarity is just shared vocabulary, not real duplication"

40-60% TF-IDF similarity between modules that aren't supposed to share concepts is a warning. Check if they're independently implementing the same logic with different helpers. The vocabulary match IS the signal.

## Relationship to Other Skills

- **indexion-plan-refactor**: Finds textual and structural duplication. Use FIRST to clear copy-paste issues.
- **indexion-refactor** (this skill): Finds concept-level duplication and enforces SoT. Use AFTER plan-refactor, or when the problem isn't copy-paste but scattered ownership.
- **indexion-plan-solid**: Extracts shared code into common packages. Complementary — solid finds extraction candidates, refactor ensures the extraction is complete.
- **indexion-explore**: The diagnostic tool. Use to measure similarity before and after consolidation.

## Example: Archive Text Extraction SoT

**Problem detected by explore**: `vfs.mbt` (50%), `discover.mbt` (50%), `args.mbt` (46%) — three files in the same concept domain.

**Analysis**: All three independently implement "check if archive, build spec, expand, skip binary, collect text". The concept "extract text from an archive" has no single owner.

**Fix**: 
1. `vfs.mbt` gains `try_extract_archive_text()` and `try_extract_archive_entries()` — the SoT
2. `discover.mbt` calls `try_extract_archive_text()` — one line
3. `args.mbt` calls `try_extract_archive_entries()` — one line
4. `is_archive_spec` and `ArchiveSpec::from_spec` are no longer called outside `@vfs`

**Verification**: `indexion explore vfs.mbt discover.mbt args.mbt` shows reduced similarity. No `is_archive_spec` calls exist outside `@vfs` (`grep` confirms). SoT test asserts `SupportedFile.path` never contains `!/`.
