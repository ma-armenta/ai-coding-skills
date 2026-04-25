---
name: codebase-cleanup
description: >
  A systematic, multi-phase codebase hardening and cleanup skill. Use this skill whenever the
  user wants to refactor, clean, or harden a codebase — including tasks like: eliminating dead
  code, removing unused exports, finding circular dependencies, strengthening weak types,
  consolidating shared types, deduplicating repeated logic (DRY), removing silent error
  swallowing, stripping AI-generated stubs and placeholder comments, and removing legacy or
  deprecated code paths. Trigger this skill for any request involving "clean up the codebase",
  "remove dead code", "find unused files", "fix circular imports", "remove any types",
  "consolidate types", "refactor duplicated logic", "remove placeholder comments", or
  "codebase hygiene". Works across all languages: TypeScript, Python, Go, Rust, Java, etc.
  Always use this skill before responding to such requests — do not attempt these phases
  ad-hoc without the structured checklist this skill provides.
---

# Codebase Cleanup & Hardening Skill

A disciplined, phased approach to codebase cleanup. Each phase is independent but sequenced
for minimum disruption. Work through all phases unless the user specifies otherwise.

**Important**: This skill is language-agnostic. Adapt tool choices to the language(s) in the
repo. The phases, principles, and checklist apply universally.

---

## Phase 0 — Orient Before Touching Anything

Before making a single change:

1. **Identify the language(s)** in the repo and available tooling.
2. **Map the entry points**: What runs? What's the main binary, server, or export surface?
3. **Check for CI/test coverage**: Are there tests you can run to catch regressions?
4. **Confirm the user's priorities**: Are some phases more urgent than others?
5. **Create a checkpoint**: Note the current state (git status, test results if any) before
   proceeding. This gives a clean rollback baseline.

> If you can run tests, do so now and record the result as the baseline.

---

## Phase 1 — DRY Consolidation (Repeated Logic)

**Goal**: Reduce copy-paste and redundant abstractions where doing so genuinely simplifies.

### Steps

1. **Scan for repeated logic**: Look for functions that appear more than once with slight
   variations — same algorithm, same query pattern, same transformation, same validation.
   Tools: `grep`, `semgrep`, language-specific AST search, or manual inspection.

2. **Evaluate each candidate honestly**:
   - Does merging reduce complexity or just line count?
   - Do the two copies serve the *same* purpose or merely *look* similar?
   - Will a shared abstraction have a single, stable reason to change?
   - **If in doubt, do not merge.** Premature DRY introduces wrong abstractions.

3. **Consolidate only what passes that test**:
   - Extract into a shared module, utility, or base class.
   - Update all callers.
   - Confirm no behavior change (run tests if available).

4. **Do not merge code that**:
   - Serves different business rules that happen to have similar shape.
   - Will diverge in the future based on context.
   - Is in different layers (e.g., API handler vs. DB layer) — similarity is coincidental.

### Checklist
- [ ] Identified repeated logic candidates
- [ ] Evaluated each — merge only where intent is truly shared
- [ ] Extracted and consolidated surviving candidates
- [ ] All callers updated, tests passing

---

## Phase 2 — Type Consolidation (Single Source of Truth)

**Goal**: Eliminate type drift caused by the same concept being defined in multiple places.

### Steps

1. **Collect all type definitions**: Search for `interface`, `type`, `class`, `struct`,
   `dataclass`, `schema`, `TypedDict`, `Zod schema`, `Pydantic model`, or equivalent
   depending on language(s). Dump them to a list.

2. **Group by concept**: Which types represent the same domain entity or data shape?

3. **Identify duplicates and drift**:
   - Types defined in multiple files for the same concept.
   - Fields that have gone out of sync between copies (one has `createdAt`, the other
     `created_at`, one has an extra optional field the other lost).
   - Duplicate enums with different members.

4. **Merge into a single source of truth**:
   - Pick the canonical location (usually a `types/`, `models/`, or `shared/` module).
   - Delete the duplicates.
   - Update all imports.

5. **Run type checks after every batch** — do not proceed until the checker is clean.

### Checklist
- [ ] All type definitions collected
- [ ] Duplicates and drift identified
- [ ] Merged to single source of truth
- [ ] Imports updated everywhere
- [ ] Type checker passes

---

## Phase 3 — Dead Code Removal

**Goal**: Remove unused exports, unreferenced functions, and orphaned files.

### Tools by Language

| Language | Static Analysis Tools |
|---|---|
| TypeScript/JS | `knip`, `ts-prune`, `eslint unused-vars` |
| Python | `vulture`, `pylint`, `pyflakes` |
| Go | `go vet`, IDE unused warnings, `deadcode` |
| Rust | `cargo check` (unused warnings built-in) |
| Java | IDE inspections, `PMD`, `SpotBugs` |
| Multi-lang | `semgrep`, manual `grep` for export names |

### Steps

1. **Run your static analysis tool(s)**. Generate a list of candidates.

2. **Manually verify every candidate before removing**. Check for:
   - **Dynamic imports**: `import(moduleName)`, `require(variable)` — static tools miss these.
   - **Config file references**: Webpack entry points, Jest config, tsconfig paths,
     framework convention files (Next.js pages/, Remix routes/, etc.).
   - **String-keyed access**: `obj['methodName']`, reflection, decorators.
   - **Code generation output**: Files generated at build time that reference "unused" symbols.
   - **Test fixtures and mocks**: Often look unused but are referenced indirectly.
   - **Public API surface**: If this is a library, "unused" exports may be the product.

3. **Remove only what is confirmed dead** after manual verification.

4. **Delete orphaned files** (files not imported anywhere and not an entry point).

### Checklist
- [ ] Static analysis run, candidate list generated
- [ ] Each candidate manually verified
- [ ] Confirmed-dead code removed
- [ ] Orphaned files deleted
- [ ] Tests still pass

---

## Phase 4 — Circular Dependency Resolution

**Goal**: Break import cycles that harm testability, maintainability, or correctness.

### Tools by Language

| Language | Tools |
|---|---|
| TypeScript/JS | `madge`, `dpdm`, `eslint import/no-cycle` |
| Python | `pydeps`, `pylint` cycle detection, `importlab` |
| Go | `go list -json` + custom graph walk |
| Rust | Cargo workspace structure (cycles are compile errors) |
| Java | `JDepend`, `ArchUnit` |

### Steps

1. **Map the full dependency graph**. Visualize it — cycles are easier to see than read.

2. **Triage cycles by impact**:
   - Does this cycle prevent isolated unit testing?
   - Does it cause initialization-order bugs at runtime?
   - Does it confuse readers of the code without adding expressiveness?
   - **Low-impact cycles** (two tightly coupled sibling modules) may not be worth breaking.
   - **High-impact cycles** (domain layer importing from infrastructure) must be broken.

3. **Break cycles by extracting shared logic into a neutral module**:
   - The neutral module has no dependency on either party.
   - Both parties depend on the neutral module, not on each other.
   - **Do not introduce new abstractions just to break the cycle** — if extracting a
     `types.ts` or `interfaces/` module is all it takes, do that.

4. **Verify the graph is cleaner** and all tests still pass.

### Checklist
- [ ] Dependency graph mapped
- [ ] Cycles identified and triaged
- [ ] High-impact cycles broken via neutral module extraction
- [ ] Graph re-verified, tests passing

---

## Phase 5 — Type Strengthening (Eliminate Weak Types)

**Goal**: Replace `any`, `unknown`, `object`, `interface{}`, `dict`, and other structural
placeholders with real types that reflect actual runtime shape.

### Steps

1. **Find all weak type usages**:
   - TypeScript: `any`, `unknown` (unless at a true boundary), `object`, `{}`, `Function`
   - Python: untyped params, `dict`, `Any` from `typing`
   - Go: `interface{}` or `any` (Go 1.18+) where a concrete type is knowable
   - Java: raw `Object`, unchecked casts

2. **Research the real type for each occurrence**:
   - What does the actual value look like at runtime? Inspect callers and producers.
   - What does the library or package return? Read its type definitions or docs.
   - What does the data source (API, DB, file) actually contain?

3. **Replace with strong types**:
   - Define an interface or type alias if the shape isn't already captured.
   - Narrow `unknown` by adding a type guard rather than casting.
   - Use discriminated unions for values that can be one of several known shapes.

4. **Preserve legitimate boundary types**:
   - `unknown` is correct at deserialization boundaries (JSON.parse, API responses before
     validation, file reads). Keep these — add a schema validation step instead of removing.
   - `any` is acceptable in rare interop situations with untyped third-party code where the
     cost of typing outweighs the benefit. Document why with a comment.

5. **Run the type checker after every batch**. Never proceed with a broken type check.

### Checklist
- [ ] All weak types located
- [ ] Each one researched — real runtime type identified
- [ ] Replaced with strong types or properly validated boundary types
- [ ] Type checker passes after each batch

---

## Phase 6 — Error Handling Cleanup

**Goal**: Remove silent error swallowing and hidden failures. Keep real error boundaries.

### The Rule

> An error handler earns its existence only if it does one or more of: recovery, logging,
> cleanup, or user-facing reporting. If it does none of these — remove it.

### Steps

1. **Find all error handling patterns**:
   - `try/catch` with empty body or just `console.log`
   - `except: pass`, `except Exception: pass`
   - `.catch(() => null)` or `.catch(() => defaultValue)`
   - `err != nil { return }` without propagating or logging
   - Fallback values that hide the fact an error occurred (`return ""`, `return []`)

2. **Evaluate each handler**:
   - Does it recover (retry, fallback to a real alternative, degrade gracefully)?
   - Does it log (with enough context to debug the problem)?
   - Does it clean up (close connections, release locks)?
   - Does it report to the user or caller (return an error, throw a typed exception)?
   - **If none of the above: remove the catch and let the error propagate.**

3. **Fix swallowed errors**:
   - Add logging before re-throwing if the error needs visibility.
   - Propagate the error to the caller if the current scope can't handle it.
   - Wrap in a typed/domain error if it crosses a layer boundary.

4. **Never introduce a silent fallback** to make tests pass or make code "not crash" —
   this hides real problems and creates subtle data corruption bugs.

### Checklist
- [ ] All try/catch and equivalent patterns located
- [ ] Each evaluated against the recovery/logging/cleanup/reporting test
- [ ] Silent swallowers removed; errors now propagate or are properly logged
- [ ] No new silent fallbacks introduced

---

## Phase 7 — Legacy, Deprecated, and AI Artifact Removal

**Goal**: Remove obsolete code paths, stubs, and noise left by incremental AI-assisted edits.

### 7A — Legacy and Deprecated Code

1. **Find legacy code**:
   - Flags like `DEPRECATED`, `TODO: remove`, `LEGACY`, `OLD_`, `_v1`, `_old`
   - Feature flags that are permanently on or permanently off
   - Code paths gated on config values that no longer exist
   - Import of deprecated library versions alongside current versions

2. **Verify obsolescence before removing**:
   - Is this referenced anywhere in production config or data?
   - Is it needed for a migration path that's still in progress?
   - Is it part of a public API contract that external consumers depend on?
   - **If uncertain, ask the user.** Do not guess.

3. **Remove only what is confirmed obsolete.**

### 7B — AI Artifacts

AI-assisted edits leave characteristic noise. Find and remove all of:

1. **Stubs and placeholder logic**:
   - Functions that `return null`, `pass`, `throw new Error("Not implemented")`
   - `# TODO: implement`, `// FIXME: add real logic`, `/* placeholder */`
   - Hardcoded test values left in production paths

2. **Edit-history comments** (comments that narrate what changed, not why):
   - `// Updated by Claude to fix X`
   - `// Changed from Y to Z`
   - `// Refactored for clarity`
   - `// Added error handling`
   - Any comment that describes a past action rather than explaining present intent

3. **Redundant or obvious comments**:
   - `// increment counter` above `counter++`
   - `// return the result` above `return result`
   - Comments that restate the code rather than explain the *why*

4. **Rewrite comments that are worth keeping**:
   - A good comment explains **why** the code exists, not what it does.
   - It should be useful to a new engineer who has never seen this codebase.
   - Rewrite: `// This handles the edge case` → `// When the user has no profile yet,
     the API returns null instead of an empty object — guard here before accessing fields`

### Checklist
- [ ] Legacy flags and deprecated paths removed (verified safe)
- [ ] Feature flag dead branches removed
- [ ] Stubs and unimplemented placeholders removed or implemented
- [ ] Edit-history comments removed
- [ ] Obvious/restatement comments removed
- [ ] Remaining comments rewritten to explain *why*, not *what*

---

## Execution Order and Batching

Run phases in this order to minimize rework:

```
Phase 0 (Orient)
  → Phase 2 (Type Consolidation)   ← before dead code, so shared types exist
  → Phase 1 (DRY Consolidation)    ← after types are stable
  → Phase 3 (Dead Code Removal)    ← after DRY, so nothing is "dead" by mistake
  → Phase 4 (Circular Deps)        ← graph is cleaner after dead code is gone
  → Phase 5 (Type Strengthening)   ← types are consolidated, safe to strengthen
  → Phase 6 (Error Handling)       ← logic is stable, safe to change error paths
  → Phase 7 (Legacy + AI Artifacts) ← cosmetic/structural last
```

**Batch commits**: Make one commit per phase (or per logical chunk within a phase).
This makes the history readable and rollbacks surgical.

**Run type checks and tests after every phase.**

---

## Language-Specific Tool Quick Reference

| Task | TypeScript | Python | Go | Rust |
|---|---|---|---|---|
| Dead exports | `knip`, `ts-prune` | `vulture` | `deadcode` | `cargo check` |
| Circular deps | `madge` | `pydeps` | `go list` graph | N/A (compile error) |
| Type coverage | `tsc --strict` | `mypy --strict` | `go vet` | built-in |
| Unused vars | `eslint` | `pylint` | `go vet` | built-in |
| Semgrep | ✓ | ✓ | ✓ | ✓ |

Install tools only if not already present. Always prefer tools already in the project's
`devDependencies` or virtual environment.

---

## What NOT To Do

- **Do not merge code that merely looks similar** — verify shared purpose first.
- **Do not remove exports without manual verification** — static tools miss dynamic usage.
- **Do not break circular deps by introducing abstractions** — extract neutral modules only.
- **Do not cast away `unknown`** — add a validator instead.
- **Do not add silent fallbacks** to make errors "go away."
- **Do not remove legacy code without confirming it's not in a live migration path.**
- **Do not rewrite working code** in the name of cleanup — fix structure, not style.