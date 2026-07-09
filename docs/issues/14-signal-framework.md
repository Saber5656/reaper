# Title

Signal framework: collector contract, orchestration, generated-file and churn collectors

## Summary

Implement the signal-collection stage: the `SignalCollector` contract, the orchestrator that runs collectors over normalized findings, and the two simple built-in collectors (S7 generated-file, S11 recent-churn) as reference implementations.

## Context

DESIGN.md §10.5 defines twelve signals produced by six collectors (issues 15–20 build the complex ones). This issue builds the chassis plus the two collectors that need no language knowledge, so the scoring engine (21) has real inputs early.

## Scope

In: `src/signals/types.ts`, orchestrator in the same module, `src/signals/generated.ts`, `src/signals/churn.ts`.
Out: string-reference (15), dynamic-usage (16), entrypoints (17–19), suppression (20), scoring math (21).

## Detailed Requirements

1. `src/signals/types.ts`:
   ```ts
   interface SignalCollector {
     id: string;                                  // e.g. "generated"
     collect(ctx: SignalContext, findings: Finding[]): Promise<SignalPatch[]>;
   }
   type SignalPatch = { fingerprint: string; signal?: Signal;    // appended
                        meta?: Partial<FindingMeta> };           // merged (OR semantics for booleans)
   type SignalContext = { repoRoot: string; config: ResolvedConfig; discovery: DiscoveryResult;
                          exec: ExecRunner; log: Logger; fileIndex: RepoFileIndex };
   ```
   Collectors are additive-only (§10.5): the orchestrator applies patches; collectors never receive mutable findings. Duplicate signal IDs per finding are rejected by the orchestrator (`E_SIGNAL_DUP` — programming error).
2. `RepoFileIndex` (built once per scan, shared by 15–19): enumerates tracked files via `git ls-files -z` (respects .gitignore, includes staged), classifies each path as `code | non-code | test | generated-candidate` using extension tables + test-path conventions (frozen tables):
   - code: `.ts .tsx .mts .cts .js .jsx .mjs .cjs .py .go`
   - test: path contains `__tests__/`, `/test/`, `/tests/`, or filename matches `*.test.*`, `*.spec.*`, `*_test.go`, `test_*.py`, `conftest.py`
   - non-code: everything else textual under 2 MiB (binary sniff: NUL byte in first 8 KiB ⇒ excluded entirely)
   - index exposes `readText(path)` with per-scan memoization and a global 256 MiB read budget (over budget ⇒ collectors receive `E_INDEX_BUDGET` and the scan continues with a warn: affected signals simply don't fire — fail-open for *report* purposes but the scorer records `signalCoverage: partial` in the artifact stats; policy engine treats partial coverage as propose-blocking: reasons += `["signal-coverage-partial"]`) — this rule is part of the safety envelope.
3. Orchestrator `collectSignals(ctx, findings)`: runs collectors sequentially in fixed order (generated, churn, string-reference, dynamic-usage, entrypoints, suppression — registry list; missing ones skipped), applies patches, returns findings + `coverage: "full" | "partial"`.
4. `src/signals/generated.ts` (S7): fires `generated.file` (−0.20) + `meta.generated=true` when the finding's file (a) matches globs `**/*_pb2.py, **/*.pb.go, **/*.gen.ts, **/*.gen.go, **/__generated__/**`, OR (b) first 5 lines contain `@generated` or match `/Code generated .* DO NOT EDIT/`, OR (c) detector marked it (`nativeType: "deadfunc-generated"`, issue 11). Evidence: the matching marker/glob.
5. `src/signals/churn.ts` (S11): one `git log --since=<14 days> --name-only --pretty=format:` pass builds the recently-touched set; findings whose file is in it get `churn.recent` (−0.10), evidence = most recent commit hash + date. Window from a module-level const `CHURN_WINDOW_DAYS = 14` (config knob deferred; documented).
6. Unit tests: orchestrator patch application + dup rejection + coverage propagation; index classification table tests; generated matrix (glob / header / detector-marker); churn with a scripted git log via fake exec.

## Acceptance Criteria

- [ ] Collector contract + orchestrator tests pass; additive-only invariant holds (original findings objects unmutated — test with frozen objects).
- [ ] Index classifies the frozen tables correctly incl. binary exclusion and budget behavior.
- [ ] S7 fires on all three trigger classes with correct evidence; S11 fires only within window.
- [ ] Read-budget breach ⇒ coverage `partial` and scan still succeeds (test).

## Validation

`pnpm test src/signals`.

## Dependencies

02, 03, 05, 06, 07 (Finding flow), 11 (generated marker convention — soft; marker string frozen here and in 11).

## Non-goals

The four complex collectors (15–20), score arithmetic (21), configurable churn window (v2).

## Design References

DESIGN.md §10.5 (S7, S11, additive-only rule), §10.1, §17 T4 (budget); ADR-003.
