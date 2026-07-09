# Title

Canonical Finding model, fingerprint function, and finding.schema.json

## Summary

Implement the canonical data model every stage exchanges: TypeScript types for Finding and the scan artifact, the stable fingerprint function, and the published JSON Schema, with golden tests.

## Context

DESIGN.md §5 freezes the Finding shape and fingerprint algorithm. Adapters produce it, signals append to it, policy decides on it, PRs dedupe by it, fixtures assert on it. Getting this wrong later means migrating every module, so it ships first with exhaustive tests.

## Scope

In: `src/core/finding.ts`, `src/core/fingerprint.ts`, `src/core/errors.ts`, `src/core/log.ts`, `schema/finding.schema.json`, ajv validation helper.
Out: any producer/consumer of findings; config schema (03).

## Detailed Requirements

1. `src/core/finding.ts`: export the exact union/interface types from DESIGN.md §5.1 — `Category` (`"unused-file" | "unused-export" | "unused-symbol" | "unused-dependency"`), `Language` (`"ts" | "js" | "python" | "go"`), `SymbolKind` (11 values listed in §5.1), `DetectorName` (5 values), `Location`, `DetectorInfo`, `Signal` (`{ id: string; delta: number; evidence: string[] }`), `Confidence` (`{ base: number; score: number; band: "high"|"medium"|"low" }`), `Decision` (`{ action: "propose"|"report"|"suppress"; reasons: string[] }`), `FindingMeta` (`{ testOnlyUsage: boolean; generated: boolean }`), `Finding`, `RawFinding = Omit<Finding, "fingerprint"|"signals"|"confidence"|"decision"|"corroboration"|"meta">`, `ScanArtifact` (§5.1 wrapper incl. `adapters[]` status entries with `status: "ok"|"skipped"|"failed"` and optional `reason`).
2. `src/core/fingerprint.ts`: `fingerprintOf(f: Pick<Finding,"category"|"workspace"|"location">): string` implementing exactly `hex(sha256("v1|" + category + "|" + workspace + "|" + location.path + "|" + (symbol ?? "") + "|" + (symbolKind ?? "")))[0:16]` using `node:crypto`. Paths must already be repo-root-relative POSIX; the function throws `ReaperError("E_FINGERPRINT_INPUT")` on absolute or backslash-containing paths.
3. `src/core/errors.ts`: `class ReaperError extends Error { code: string; hint?: string }` plus the frozen exit-code map from DESIGN.md §7 as `EXIT_CODES` const.
4. `src/core/log.ts`: leveled logger (error/warn/info/debug) writing to **stderr** only, `--no-color` aware, with `scrub(s: string)` applied to every line: redacts strings matching `/\b(gh[pousr]_[A-Za-z0-9_]{20,}|github_pat_[A-Za-z0-9_]{20,})\b/g` to `***` (DESIGN.md §17 T2).
5. `schema/finding.schema.json`: JSON Schema draft 2020-12 for the ScanArtifact (embedding Finding), `additionalProperties: false` everywhere, numeric ranges (`score`/`base` in [0,1], `delta` in [-1,1]), enum constraints matching the TS types. `src/core/finding.ts` exports `validateScanArtifact(json: unknown)` using ajv compiled once.
6. Golden tests (`src/core/fingerprint.test.ts`): at least 8 fixed input→hash vectors covering each category, symbol-less findings, unicode symbol names, nested workspace; a test asserting line numbers do NOT affect the fingerprint; error cases (absolute path, backslash).
7. Schema round-trip test: a hand-written maximal ScanArtifact fixture validates; mutations (unknown key, score 1.5, bad enum) each fail with pointer to the offending path.

## Acceptance Criteria

- [ ] Types compile under strict mode and are the single import source for all model types (`src/core/finding.ts`).
- [ ] Fingerprint golden vectors pass and are committed (they are the compatibility contract).
- [ ] `validateScanArtifact` accepts the maximal fixture and rejects each mutation class with a precise error.
- [ ] Logger writes only to stderr; token-shaped strings are scrubbed (test with a fake `github_pat_…`).
- [ ] `schema/finding.schema.json` is valid JSON Schema (ajv compiles it without warnings).

## Validation

`pnpm test src/core` green. Manually run `node -e` computing one fingerprint and compare against the documented vector in the test file.

## Dependencies

01-project-bootstrap.

## Non-goals

Scoring math (21), decision logic (22), artifact file I/O location conventions (31).

## Design References

DESIGN.md §5 (model + fingerprint), §7 (exit codes), §17 T2 (scrubbing).
