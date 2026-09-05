# Title

Detector adapter contract, registry, and adapter conformance test kit

## Summary

Implement the `DetectorAdapter` interface, the registry that resolves enabled adapters per language root, normalization from RawFinding to Finding (fingerprint + corroboration merge), and a reusable conformance test kit that every adapter (08–12) must pass.

## Context

DESIGN.md §9.1 freezes the adapter contract; ADR-002 makes it the language-extension seam. Building the conformance kit before any concrete adapter keeps all five adapters behaviorally uniform (defensive parsing, ExecRunner-only spawning, skip semantics).

## Scope

In: `src/adapters/types.ts`, `src/adapters/registry.ts`, normalization + corroboration merge, `test/kit/adapter-conformance.ts`.
Out: the five concrete adapters (08–12), doctor UI (13).

## Detailed Requirements

1. `src/adapters/types.ts`: `DetectorAdapter`, `AdapterContext`, `AdapterRunStatus = { name: DetectorName; version?: string; status: "ok"|"skipped"|"failed"; reason?: string }` exactly per DESIGN.md §9.1 (RawFinding type imports from core, issue 02).
2. `src/adapters/registry.ts`:
   - `const ALL_ADAPTERS: DetectorAdapter[]` (populated by 08–12; starts empty-importable).
   - `selectAdapters(cfg, roots, only?: DetectorName[]): { adapter, root }[]` — cross-product of enabled languages × adapter.languages × discovered roots, filtered by `--only`.
   - `runAdapters(ctx, pairs): Promise<{ raw: RawFinding[]; statuses: AdapterRunStatus[] }>` — sequential per root (parallelism is a v2 concern; determinism first), each adapter call wrapped: `probe()` first (unavailable ⇒ status `skipped/tool-missing` with `hint`; with `--strict-tools` ⇒ `ReaperError("E_TOOL_MISSING_STRICT")` exit 4); `run()` exceptions ⇒ status `failed` with reason = error message head (200 chars), never a crash.
3. Normalization (`normalizeFindings(raw: RawFinding[]): Finding[]`):
   - Compute fingerprint (02); initialize `signals: []`, `corroboration: []`, `confidence` zeroed placeholder, `decision` placeholder `{action:"report",reasons:[]}`, `meta` defaults.
   - **Corroboration merge:** identical fingerprints from different detectors collapse into one Finding — keep the detector with the higher base score (per §10.3 table exported from confidence module as data; until issue 21 lands, use a local literal copy marked `// KEEP IN SYNC §10.3`), append others' names to `corroboration[]`.
   - Path hygiene: all `location.path` values normalized to repo-root-relative POSIX; adapter-relative paths are joined with the root dir; absolute paths inside the repo are relativized; paths escaping the repo root ⇒ that finding is dropped with a warn log (T3 pre-filter).
4. Conformance kit (`test/kit/adapter-conformance.ts`): exported `describeAdapterConformance(adapter, fixtures)` running against a fake ExecRunner:
   - probe() never throws when the binary is missing (fake runner returns ENOENT-like failure).
   - run() with malformed tool output (truncated JSON / garbage lines) ⇒ throws typed `E_ADAPTER_PARSE` (registry maps to `failed`), never returns partial silent results.
   - run() emits only paths inside the given root; no absolute paths; symbolKind within enum.
   - run() performs zero writes inside the repo tree (fake fs sentinel: kit snapshots mtimes of a fixture dir).
   - every spawn goes through the provided `ctx.exec` (kit's fake counts calls; direct `child_process` is already lint-banned by 05).
5. Fake ExecRunner test double (`test/kit/fake-exec.ts`): scriptable `{ match: (req) => bool, result: ExecResult }[]` — shared by all adapter tests.

## Acceptance Criteria

- [ ] Registry + normalization unit tests pass, including a synthetic corroboration case (two fake adapters, same fingerprint → one Finding, `corroboration: ["other"]`, higher-base detector kept).
- [ ] Escaping-path RawFinding is dropped with a warning (test).
- [ ] `--strict-tools` with a missing tool exits 4; without it, status `skipped` and scan continues (integration-style test with fake adapters).
- [ ] Conformance kit exported and demonstrated against a minimal fake adapter in its own test.

## Validation

`pnpm test src/adapters test/kit` green.

## Dependencies

02, 03, 05, 06.

## Non-goals

Concrete adapters (08–12), parallel adapter execution (v2), scoring (21).

## Design References

DESIGN.md §9.1 (contract), §9.2 (matrix), §10.3 (base table sync note), §17 T3/T4; ADR-002.
