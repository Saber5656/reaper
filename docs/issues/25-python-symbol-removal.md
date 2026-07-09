# Title

Python symbol removal strategy via bundled LibCST codemod

## Summary

Implement the `py-symbol` removal strategy per DESIGN.md §12.3: a bundled Python LibCST script that deletes function/class/assignment declarations named in a JSON plan, invoked from the TS strategy via uv/python3, with line-tolerance guards and `__all__` cleanup.

## Context

Python `unused-symbol` findings (vulture) are report-by-default, propose-eligible via config. Node cannot rewrite Python safely; a small LibCST codemod, exact-pinned and invoked as a subprocess, keeps the removal correct and comment-preserving.

## Scope

In: `src/removal/py-symbols.ts` (TS side), `src/removal/py/remove_symbols.py` (codemod), tests for both.
Out: unused imports (never — linter territory), Python file deletion (v2), whitespace re-formatting.

## Detailed Requirements

1. TS side `py-symbols.ts`:
   - `canHandle`: `category === "unused-symbol"` && language python && symbolKind ∈ {function, method, class, variable}.
   - Plan JSON to stdin: `{ "version": 1, "edits": [ { "file": "<worktree-relative>", "symbol": "name", "kind": "function|method|class|variable", "line": <int> } ] }`.
   - Invocation resolution (mirrors 09): `uv run --with libcst==<LIBCST_PIN> python3 <abs path to bundled remove_symbols.py>` — the script ships inside reaper's npm package (`files` includes `src/removal/py/`); fallback `python3` if `import libcst` probe succeeds; neither ⇒ whole-strategy per-finding failures `tool-missing:libcst`.
   - Every plan file passes `assertSafeTarget` (23) BEFORE invocation; script also receives `--root <worktree>` and re-validates containment (defense in depth).
   - Result JSON from stdout: `{ "results": [ { "file", "symbol", "status": "removed|not-found|line-mismatch|ambiguous|error", "detail"? } ] }` mapped to per-finding success/failure.
2. Codemod `remove_symbols.py` (single file, stdlib + libcst only):
   - Parses plan from stdin; per edit: locate `FunctionDef`/`ClassDef`/top-level `Assign`/`AnnAssign` whose name matches AND whose `def`/decl line is within ±2 of `line` (shadowing guard §12.3); method lookup: `Class.method` composite symbols split and matched inside the class body.
   - Multiple same-name matches within tolerance ⇒ `ambiguous`, no edit (safety).
   - Removal includes decorators and the declaration's leading comment lines that are contiguous (no blank line between comment block and decl).
   - `__all__` entries: any string element equal to a removed top-level symbol is removed from `__all__` lists in the same file.
   - Writes files atomically (tmp + rename) only under `--root`; prints result JSON; exit 0 even with per-edit failures (protocol errors exit 2).
3. Determinism: edits applied per file in descending line order.
4. Tests:
   - Codemod: pytest-style golden tests runnable via `uv run --with libcst --with pytest pytest src/removal/py/` wired into CI as a separate job step (documented in the issue PR); cases — function/method/class/variable removal, decorator + leading-comment inclusion, ±2 tolerance, ambiguity refusal, `__all__` cleanup, nested same-name shadowing untouched, non-existent file → error entry.
   - TS side: fake-exec protocol tests (plan serialization, result mapping, tool-missing path, containment pre-check).
5. `LIBCST_PIN` recorded next to the other pins (central `src/adapters/pins.ts` — create it here if 08–12 haven't; coordinate).

## Acceptance Criteria

- [ ] Codemod golden suite green under pinned libcst in CI (Linux + macOS).
- [ ] Ambiguity and line-mismatch produce NO edit and a machine-readable status (tests).
- [ ] Removed decorated function takes its decorators and contiguous leading comment with it; blank-line-separated comments survive (tests).
- [ ] TS-side maps every status to per-finding results; batch isolation holds (one ambiguous + one clean ⇒ 1 changed, 1 failed).
- [ ] `python3 -m compileall` passes on all codemod outputs in tests (pre-verification of what 27 will assert).

## Validation

`pnpm test src/removal/py-symbols*` + the pytest job step; record both outputs in the PR.

## Dependencies

23 (chassis), 09 (finding shapes + uv resolution pattern).

## Non-goals

Import cleanup in other files (nothing imports these by claim; verify catches), fixing now-unused imports the removal creates in the SAME file (report-next-run), Python 2 syntax.

## Design References

DESIGN.md §12.3 (python), §12.1, §13.2 (compileall), §17 T5 (pinning); ADR-002/ADR-003.
