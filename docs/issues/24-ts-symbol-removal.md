# Title

TS/JS symbol removal strategy via ts-morph (export deletion / demotion / re-export pruning)

## Summary

Implement the `ts-symbol` removal strategy per DESIGN.md §12.3: delete unreferenced exported declarations, demote internally-referenced ones to local, prune re-export specifiers — all inside the worktree, verified parseable afterward.

## Context

`unused-export` is report-by-default but propose-eligible via config (§11); the strategy must exist for the envelope to be honest. knip's claim is "the *export* is unused" — the minimal correct edit differs by whether the declaration is used internally (§12.3 demotion semantics, K7).

## Scope

In: `src/removal/ts-symbols.ts` + `ts-morph` dev-of-runtime dependency (lazy dynamic `import()` so scan-only runs never load it).
Out: file deletion (23), knip detection semantics (08), organize-imports/formatting (never — minimal diffs).

## Detailed Requirements

1. Add `ts-morph` exact-pinned runtime dependency; loaded lazily inside `apply` only (startup-cost + dependency-surface control; DESIGN.md §3.3).
2. `canHandle`: `category === "unused-export"` && language ts/js && symbolKind ∈ {export, function, variable, type, interface, enum, class}.
3. `apply` procedure per finding (project loaded once per batch with `useInMemoryFileSystem: false`, rooted at the worktree; tsconfig discovered from the finding's workspace, fallback default compiler options):
   a. Locate the declaration in `location.path` by name + kind + `startLine` tolerance ±2 (guards drift); not found ⇒ per-finding failure `symbol-not-found`.
   b. Case analysis:
      - Re-export specifier (`export { x } from "./y"` / `export { x }`): remove the specifier; remove the whole statement if it becomes empty.
      - Declaration with internal references in the same file (ts-morph `findReferencesAsNodes` scoped to the file): remove only the `export` modifier (demotion, §12.3/K7); record `edit: "demoted"`.
      - Declaration with zero internal references: remove the declaration node entirely; record `edit: "deleted"`.
      - `export default` (named or anonymous): treated as declaration removal of the default export statement.
   c. Path safety: every touched file passes `assertSafeTarget` (23).
   d. Post-edit: file-level syntax check via ts-morph diagnostics (syntactic only); any syntactic diagnostic ⇒ revert that file's edit (keep pre-edit text in memory), per-finding failure `postcheck-syntax`.
4. Multi-finding batches touching the same file are applied through one Project instance in `startLine` descending order (edit stability).
5. `RemovalResult.changedFiles` = unique saved files; result detail per finding records `edit` kind (surfaced in PR body table by 29 — coordinate a `detail?: Record<fingerprint,string>` extension of RemovalResult in 23's types now).
6. Tests (fixture TS files in temp worktrees): each case-analysis branch; ±2 line drift accepted, 5-line drift refused; same-file multi-edit ordering; syntax-broken postcheck revert; namespace re-export (`export * from`) findings ⇒ per-finding failure `unsupported-reexport-star` (explicit, not silent).

## Acceptance Criteria

- [ ] All case branches + failure modes covered; demotion leaves the declaration compiling as local.
- [ ] Postcheck revert restores byte-identical original on induced syntax breakage.
- [ ] ts-morph is absent from the module graph for `reaper scan` (test: spy on dynamic import / verify lazy path).
- [ ] End-to-end (with 27/31 later): a proposed unused-export batch survives `tsc --noEmit` on the ts fixture — assertion tracked in 33.

## Validation

`pnpm test src/removal/ts-symbols*`.

## Dependencies

23 (chassis + path safety + result extension), 08 (finding shapes).

## Non-goals

`export * from` handling (explicit unsupported failure), cross-file dead-chain cascade (future scans catch), formatting/organize-imports, JS without tsconfig beyond default options.

## Design References

DESIGN.md §12.3 (semantics incl. demotion rationale), §12.1, §19 K7; ADR-003 (minimal-edit principle).
