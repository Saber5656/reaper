# Title

Go entry-point & public-surface heuristics (S6/S8)

## Summary

Implement the Go entry-point pack: runtime/tooling-invoked functions (S6, −0.60) and exported-identifier library surface (S8, −0.30 for libraries), per DESIGN.md §10.5.

## Context

deadcode's RTA already handles `main`/`init` reachability, but tool-invoked and cgo/linkname edges escape it, and for library-ish modules every exported identifier is potential external API. Go findings are report-only in v1, but their *scores* still drive report ranking and future v2 removal — the heuristics must be right now.

## Scope

In: `src/signals/entrypoints/go.ts` + tests (pack contract from 17).
Out: TS/Python packs, Go removal (v2).

## Detailed Requirements

1. Rule table `GO_ENTRYPOINT_RULES`. S6 fires when:
   - Symbol is `main` or `init` in any package (belt-and-braces; deadcode shouldn't report them, but a spoofed/drifted tool output must not score high — defense against T-class parser trust issues).
   - Declaration region (5 lines above `startLine`, comment-aware via lex.ts) contains: `//go:linkname`, `//export ` (cgo), `//go:cgo_export_dynamic`, or a build-constraint line `//go:build` combined with non-default tags (evidence `buildtag:<expr>`; rationale: deadcode ran under default tags only — issue 11 non-goal — so tag-gated code is unproven, not dead).
   - Test-framework conventions: symbols matching `^Test[A-Z]`, `^Benchmark[A-Z]`, `^Fuzz[A-Z]`, `^Example[A-Z]?` in `*_test.go` files.
   - File conventions: `main.go` in a `package main` dir (whole-file findings — n/a in v1 but rule kept for v2), generated-adjacent `//go:generate`-declaring files are NOT entry points (explicit non-rule; documented).
2. S8 (repoKind=library): finding's symbol is exported (first rune uppercase, receiver-stripped: for `(*Client).Close` test `Close` and `Client`) ⇒ S8, evidence `exported-identifier`. Not gated on import-path reachability in v1 (approximation documented).
3. Implementation notes: pure textual + index reads; receiver parsing shared with adapter 11 (`export parseGoSymbol(name)` from a small `src/signals/entrypoints/go-symbol.ts`, refactor 11 to import it — or duplicate with sync comment if 11 landed first; prefer the shared helper).
4. Tests: rule matrix firing/non-firing per rule id (linkname region boundary; TestX in non-test file must NOT fire; `Example` bare name fires; lowercase symbol no S8; exported method receiver handling); repoKind gating.

## Acceptance Criteria

- [ ] Every rule id covered both ways; receiver-qualified symbols handled (`(*T).M` → checks `M`/`T`).
- [ ] Build-tagged file findings get S6 with the tag expression in evidence.
- [ ] Library module: exported dead func lands ≤ 0.45 after S8 (integration check with 21, mirroring DESIGN.md §10.6#3).
- [ ] Application module: same finding stays 0.75 base (no S8).

## Validation

`pnpm test src/signals/entrypoints/go*`.

## Dependencies

14, 17 (contract), 11 (symbol-format alignment).

## Non-goals

Import-graph-based external-usage proof (v2), cgo body analysis, assembly (`.s`) reference scanning (K-class unknown; document if fixtures hit it).

## Design References

DESIGN.md §10.5 S6/S8, §10.6#3, §9.6 (build-tag limitation), §8 (repoKind); ADR-003.
