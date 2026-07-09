# Title

Confidence scoring engine: base tables, vulture mapping, corroboration, clamping, bands

## Summary

Implement the deterministic scorer: `score = clamp(base + Σ deltas, 0.02, 0.99)` with the frozen base-score table, vulture native-confidence mapping, corroboration bonus S10, and band assignment, per DESIGN.md §10.

## Context

This is the arithmetic heart of ADR-003. Everything upstream (adapters, signals) produces inputs; everything downstream (policy, reports, PR bodies, explain) consumes its output. It must be pure, table-driven, and reproducible to the cent.

## Scope

In: `src/confidence/score.ts`, `src/confidence/bands.ts` + tests; removal of the temporary base-table copy in 07's normalizer (single source here).
Out: decisions (22), any signal collection.

## Detailed Requirements

1. `src/confidence/score.ts` exports:
   - `BASE_SCORES`: the exact table from DESIGN.md §10.3 keyed by `(detector.name, nativeType-class)` — knip nativeTypes group as: `files→0.80`, `exports|types→0.70`, `enumMembers|classMembers→0.60`, `dependencies|devDependencies→0.75`; `deptry DEP002→0.75`; `go-deadcode deadfunc|deadfunc-generated→0.75`; `go-mod-deps tidy-diff→0.90`; vulture → formula below. Unknown (detector, nativeType) ⇒ `ReaperError("E_SCORE_UNKNOWN_BASE")` — a programming error, never silent 0.
   - `vultureBase(nativeConfidence)`: `round2(0.45 + (c − 60) / 40 * 0.35)` per §10.4; input outside 60–100 clamps into range with a warn.
   - `scoreFinding(f: Finding): Finding` (pure, returns new object): base per above; S10 `corroboration.agreement` (+0.10) appended by the scorer itself when `corroboration.length ≥ 1` (§10.5 — the only scorer-emitted signal); sum all `signals[].delta`; `score = clamp(round2(base + sum), 0.02, 0.99)`; zero-delta signals participate as 0.
   - All floating math in integer cents internally (base 80 + deltas −40 …) to guarantee cross-platform determinism; convert to 2-dp floats at the boundary.
2. `src/confidence/bands.ts`: `bandOf(score)` — high ≥ 0.80, medium ≥ 0.50, low otherwise (§10.2); exported constants `HIGH_FLOOR = 0.80` reused by config validation (03 refactor: import instead of literal).
3. Determinism: given identical finding inputs, output is byte-identical; signals order must not matter (sum-commutativity test with shuffled signals).
4. Golden worked-example tests reproducing DESIGN.md §10.6 exactly:
   1. knip unused-file, no signals → 0.80 high.
   2. vulture 60% + S1(−0.40) → 0.05 low.
   3. deadcode 0.75 + S8(−0.30) → 0.45 low.
   4. knip dep 0.75, no corroboration → 0.75 medium (not proposable).
   Plus: corroborated pair (0.75 + 0.10 = 0.85 high); clamp floor (base 0.45 + S1 + S6 → 0.02); clamp ceiling (0.90 + 0.10 → 0.99); unknown base error.
5. `signalCoverage` passthrough: scorer copies the orchestrator's coverage flag into artifact stats (type plumbing; consumed by 22).

## Acceptance Criteria

- [ ] All §10.6 worked examples reproduced to the exact 2-dp values in tests referencing the design section numbers.
- [ ] Shuffle-invariance and integer-cent determinism tests pass.
- [ ] 07's temporary base-table copy deleted; single source proven by grep test (`KEEP IN SYNC` marker gone).
- [ ] Unknown (detector, nativeType) throws `E_SCORE_UNKNOWN_BASE` (test).
- [ ] `bandOf` boundary tests: 0.7999→medium is impossible in 2-dp domain; 0.80→high, 0.79→medium, 0.50→medium, 0.49→low.

## Validation

`pnpm test src/confidence`; `reaper explain` output spot-checked once 31 lands.

## Dependencies

02, 07 (normalizer refactor), 14 (Signal shape). Signals 15–20 are consumers' concerns — scorer only needs the Signal contract.

## Non-goals

Weight tuning (fixture-driven, ongoing per ADR-003), ML re-ranking (v2), policy thresholds (22).

## Design References

DESIGN.md §10.1–§10.6 (entire section is the spec); ADR-003.
