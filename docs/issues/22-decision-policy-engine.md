# Title

Decision policy engine: suppress/propose/report with machine-readable reasons

## Summary

Implement `src/policy/decide.ts`: the ordered rule evaluation from DESIGN.md §11 mapping scored findings + config + dismissed-set + capability matrix to `decision.action` with exhaustive `reasons[]`.

## Context

This is where the safety envelope becomes enforceable logic. Every "why did/didn't reaper act" question must be answerable from `reasons[]` — it powers `explain`, report columns, and PR provenance.

## Scope

In: policy module + the removal-capability matrix as data + tests.
Out: score computation (21), dismissed-set fetching (29 — injected here as a plain `Set<string>`), publication skipping for already-open PRs (29 handles at publish time; policy stays pure).

## Detailed Requirements

1. Signature: `decide(f: Finding, ctx: PolicyContext): Finding` (pure). `PolicyContext = { config: ResolvedConfig; dismissed: Set<string>; signalCoverage: "full"|"partial" }`.
2. Removal-capability matrix as exported data (single source; consumed by 23–26 registration asserts and by reports):
   ```ts
   const REMOVABLE: Record<Category, Partial<Record<Language, RemovalStrategyId>>> = {
     "unused-file":       { ts: "file-delete", js: "file-delete" },
     "unused-export":     { ts: "ts-symbol",  js: "ts-symbol" },
     "unused-symbol":     { python: "py-symbol" },
     "unused-dependency": { ts: "dependency", js: "dependency", python: "dependency", go: "dependency" },
   };
   ```
3. Ordered evaluation exactly per §11 (first match wins), each step appending its reason literal:
   1. suppress: S9 present (`suppression.keep` / `suppression.protect`) → reason = the signal id; fingerprint ∈ dismissed && !retryDismissed → `dismissed-by-closed-pr`; `actions[category] === "off"` → `category-off`.
   2. drop: `score < reportThreshold` → action `report` with reason `below-noise-floor` AND a `dropped: true` flag consumed by renderers (findings stay in the artifact for stats; excluded from human reports) — add optional `decision.dropped?: boolean` to the core type (02 amendment, coordinated).
   3. propose requires ALL, else fall to 4 (every failed predicate appended as a reason so near-misses are explainable):
      `band === "high"`, `score >= proposeThreshold`, `actions[category] === "propose"`, `REMOVABLE[category][language]` exists, `!meta.testOnlyUsage`, `!meta.generated`, `signalCoverage === "full"`.
      All pass ⇒ action `propose`, reasons = the satisfied-predicate summary `["band=high","score=0.86>=0.80","policy=propose","strategy=file-delete"]`.
   4. report with collected reasons (e.g. `["band=high","policy=report:unused-export"]` or `["no-removal-strategy:go/unused-symbol"]`, `["test-only-usage"]`, `["signal-coverage-partial"]`).
4. Reason literals are frozen contract strings (renderers and fixtures match on them) — export `REASONS` const enum-like map; free-text only after a `:` suffix.
5. Batch API `decideAll(findings, ctx)` + artifact stats fill (`stats.byAction`, `byBand`, `byCategory`).
6. Tests: full decision table — for each rule a minimal finding; the seven propose predicates each individually failing (parameterized); dismissed with/without retryDismissed; partial coverage blocking propose; drop-floor behavior; determinism (same input twice ⇒ deep-equal).

## Acceptance Criteria

- [ ] Decision-table tests cover every branch and every reason literal (100% branch coverage on decide.ts enforced via vitest coverage threshold for this file).
- [ ] DESIGN.md §10.6#4 walkthrough (medium dep not proposed despite propose policy) reproduced.
- [ ] A propose decision is impossible for any (category, language) outside REMOVABLE (property-style test iterating the full cross-product).
- [ ] `proposeThreshold` raised to 0.9 in config blocks a 0.85 finding with reason `score=0.85<0.90` (test).

## Validation

`pnpm test src/policy` with per-file coverage gate.

## Dependencies

02 (type amendment), 03, 21.

## Non-goals

Budget/batch slicing (29), publication-time already-open skip (29), auto-escalation of report→propose over time (v2 idea).

## Design References

DESIGN.md §11 (spec), §9.2 (matrix), §10.2, §10.6#4; ADR-003.
