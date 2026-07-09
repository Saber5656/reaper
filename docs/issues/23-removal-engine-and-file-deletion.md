# Title

Removal engine core: strategy contract, path-safety module, file-deletion strategy

## Summary

Implement the removal chassis per DESIGN.md §12.1/§12.2: the `RemovalStrategy` contract, the central path-safety module every strategy must pass through, and the first strategy (file deletion for TS/JS `unused-file`), with hostile-fixture tests.

## Context

This is the only subsystem that destroys user code, and T3 (path traversal/symlink escape) lives here. The safety module is written once, tested against hostile inputs, and reused by 24–26.

## Scope

In: `src/removal/types.ts`, `src/removal/path-safety.ts` (new file — add to DESIGN.md §4 layout on landing), `src/removal/file-delete.ts`, strategy registry.
Out: symbol strategies (24/25), dependency strategy (26), worktree creation (28), verify (27).

## Detailed Requirements

1. `src/removal/types.ts`: `RemovalStrategy`, `RemovalResult` exactly per §12.1; `RemovalStrategyId = "file-delete" | "ts-symbol" | "py-symbol" | "dependency"`; registry `getStrategy(id)`; startup assert: every id in policy's `REMOVABLE` matrix (22) resolves to a registered strategy (fail-fast wiring check).
2. `src/removal/path-safety.ts` — `assertSafeTarget(worktreeRoot: string, relPath: string, cfg): string` returning the absolute path or throwing `E_UNSAFE_PATH`:
   - reject absolute inputs, `..` segments, backslashes, NUL bytes, empty;
   - `lstat`: reject if any ancestor segment under the worktree or the target itself is a symlink (walk segments; §12.1);
   - `realpath(dirname)` containment check against `realpath(worktreeRoot)`;
   - reject `.git` prefix and `config.protect` glob matches (double enforcement — policy already suppressed these; strategies must not trust upstream);
   - reject paths outside the finding's `workspace` subtree (parameter).
3. `src/removal/file-delete.ts`:
   - `canHandle`: `category === "unused-file"` && language ts/js.
   - `apply`: per finding — assertSafeTarget; `fs.unlink`; stage via git layer later (28 owns git; this strategy only unlinks — the pipeline stages `git add -A` at commit time, 31). Missing file ⇒ per-finding failure `already-absent` (not fatal to batch, §12.1).
   - Post-condition: returns changedFiles (deleted paths).
4. Per-finding failure isolation (§12.1): strategy-level try/catch per finding; failures collected into `RemovalResult.failed` and the pipeline (31) excludes them from the batch/PR.
5. Hostile-fixture tests (temp dirs): symlinked file refused; symlinked intermediate dir refused; `../escape` refused; `.git/hooks/x` refused; protect-glob refused; workspace-escape refused; NUL/absolute/backslash refused; happy path deletes and reports; one-bad-one-good isolation.
6. Design-doc sync: add `path-safety.ts` to DESIGN.md §4 module layout in the same PR (docs-consistency requirement).

## Acceptance Criteria

- [ ] Every hostile case above throws `E_UNSAFE_PATH` with the offending path in the message; happy path green.
- [ ] Registry/matrix wiring assert fails the build if a REMOVABLE id lacks a strategy (test with a stubbed registry).
- [ ] Strategy never touches git or anything outside the given worktreeRoot (fs sentinel test).
- [ ] Mutation isolation: 3 findings, middle one hostile ⇒ result has 2 changedFiles + 1 failed.

## Validation

`pnpm test src/removal`; hostile fixtures committed under `test/hostile/` for reuse by 24–26.

## Dependencies

02, 03, 22 (REMOVABLE ids).

## Non-goals

Deleting Python/Go files (v1 matrix), pruning now-empty dirs (git handles), import-statement cleanup in referencing files (nothing references these files by definition; verify gate catches surprises).

## Design References

DESIGN.md §12.1/§12.2, §17 T3, §9.2; ADR-005.
