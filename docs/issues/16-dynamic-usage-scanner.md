# Title

Dynamic-usage scanner: per-language reflection/dispatch patterns producing S4/S5

## Summary

Implement the collector that detects dynamic-dispatch capability near a finding — patterns that make static unreachability claims unreliable — and applies the proximity-scoped penalties S4 (same file, −0.25) / S5 (same package, −0.15) per DESIGN.md §10.5.

## Context

Reflection and string-keyed dispatch are why naive dead-code removal breaks production. The string verifier (15) catches *references to the symbol*; this scanner catches *capability for invisible references* near the declaration, penalizing confidence even when no textual hit exists.

## Scope

In: `src/signals/dynamic-usage.ts` + pattern tables + tests.
Out: repo-wide dynamic penalty (deliberately proximity-scoped; a global penalty would flatten all scores), framework entry points (17–19).

## Detailed Requirements

1. Pattern tables (frozen; regex over code-class files from the index, comments/strings stripped via the shared lightweight lexer from 15 — extract that lexer into `src/signals/lex.ts` here and refactor 15 to use it):
   - TS/JS: `require(` with non-literal arg, dynamic `import(` with non-literal arg, `eval(`, `new Function(`, bracket property access with non-literal key on `exports|module.exports|this|globalThis`, `Reflect.get|apply|construct`, `window[`/`global[`.
   - Python: `getattr(`, `setattr(`, `globals()[`, `locals()[`, `importlib.import_module(`, `__import__(`, `eval(`, `exec(`, `operator.attrgetter(`.
   - Go: import of `reflect`, `plugin`, or `unsafe` in the file/package (import-block parse, not regex-in-comments).
   Each table row: `{ id, language, regex, description }` — description lands in evidence.
2. Scope resolution per finding:
   - S4 `dynamic-usage.same-file` (−0.25): any pattern hit in `location.path`.
   - S5 `dynamic-usage.same-package` (−0.15): hit in another code file within the same package dir — TS/JS & Python: same directory; Go: same package directory. S4 and S5 are mutually exclusive; strongest (S4) wins (§10.5).
   - Eligible categories: `unused-export`, `unused-symbol` (files/deps excluded: file-level dynamic import is S1/S2's job via basename; dep dynamic use is out of static reach).
3. One scan pass: pattern matching runs per FILE once (memoized per scan into `Map<path, PatternHit[]>`), findings then join against it — never re-scan per finding.
4. Evidence: up to 3 `path:line pattern-id` entries.
5. False-positive posture: patterns only ever LOWER confidence — over-matching is safe, under-matching is not; when in doubt a pattern stays in the table (mirror of 15's S2 note; document in module JSDoc).
6. Tests: per-language pattern matrix (each row fires; literal-arg `import("./x")` does NOT fire; commented-out `getattr` does NOT fire — lexer strip verified); scope matrix (same-file vs sibling-file vs unrelated-dir); exclusivity S4>S5; memoization (instrumented single pass per file).

## Acceptance Criteria

- [ ] Every pattern-table row covered by a firing and a non-firing test.
- [ ] Comments/strings never trigger patterns (lexer integration test per language).
- [ ] S4/S5 exclusivity and category eligibility enforced.
- [ ] Go `reflect` import in the same package lowers a deadcode finding by exactly 0.15 in an end-to-end scoring test (after 21; cross-referenced there).

## Validation

`pnpm test src/signals/dynamic-usage* src/signals/lex*`.

## Dependencies

14, 15 (lexer extraction refactor).

## Non-goals

Type-aware dataflow (v2/never), decorators-as-dynamic (that's entry-point territory, 17–18), penalizing the whole repo for one `eval`.

## Design References

DESIGN.md §10.5 S4/S5; §18.1 traps (getattr/registry fixtures must survive); research survey §4; ADR-003.
