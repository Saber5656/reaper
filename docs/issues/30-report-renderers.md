# Title

Report renderers: canonical JSON artifact I/O, Markdown report, GitHub Step Summary, explain command

## Summary

Implement `src/report/` per DESIGN.md §16: scan-artifact write/read with schema validation, the deterministic Markdown report, step-summary output, plus the `reaper report` and `reaper explain` command bodies.

## Context

Report-mode is the default product experience (Action default `mode: report`) and the triage surface for everything the safety envelope declines to auto-fix — the "why not auto-fixed" column is how users learn to trust the tool.

## Scope

In: `src/report/{json.ts,markdown.ts,step-summary.ts}`, command bodies for `report` and `explain` (04's stubs), artifact location convention.
Out: scan pipeline (31), PR bodies (29).

## Detailed Requirements

1. `json.ts`: `writeScanArtifact(path, artifact)` (validate via 02's `validateScanArtifact` BEFORE write — reaper never emits an invalid artifact; pretty-printed, key-sorted for diffability) and `readScanArtifact(path)` (validate after read; version mismatch ⇒ `E_ARTIFACT_VERSION`). Default location `.reaper/scan.json`; `--json <path>` additionally writes a copy (`-` = stdout, §7 purity rule).
2. `markdown.ts` — deterministic layout (§16):
   - Header: repo, timestamp, reaper version, config hash prefix.
   - Adapter status table (name, version, status, reason/stats incl. `testOnlyDead` from 11).
   - **Proposed** section: table `Path | Symbol | Category | Lang | Score | Detector` (these became/will become PRs; annotated `already-proposed:#N` / `deferred-by-budget` when applicable).
   - **Reported** section: same columns + `Band | Why not auto-fixed` (first non-satisfied propose predicate from `decision.reasons`, humanized via a fixed reason→text map covering every 22 REASONS literal).
   - **Suppressed**: count by reason class only (keep/protect/dismissed) — no table.
   - Row caps: 200 per section with explicit `… and N more (see JSON artifact)` (T4).
   - Stats footer: counts by band/category/action + `signalCoverage`.
3. `step-summary.ts`: append the Markdown to `$GITHUB_STEP_SUMMARY` when env set (write errors ⇒ warn, never fail the run).
4. `report` command: read artifact (default path or `--scan`), render to `--md <path>` / `--json <path>` / stdout summary (short: counts + top 10 by score); missing artifact ⇒ exit 2 with hint "run reaper scan first".
5. `explain <fp>` command: locate finding; print: header (path/symbol/category/detector\@version), score arithmetic table (base + each signal: id, delta, evidence lines), band, decision + every reason humanized, fingerprint. Missing fp ⇒ exit 2 listing 5 closest prefixes (Levenshtein-lite prefix match).
6. All rendering pure: `render(artifact, opts) → string` — command bodies only do I/O (testability).
7. Tests: golden snapshots for markdown (fixed artifact fixture covering all sections/caps/annotations) and explain output; artifact write-validate-read round-trip; version-mismatch; step-summary env behaviors; stdout purity with `--json -` (logs on stderr only); reason→text map completeness test (iterates 22's REASONS export — a new reason literal without a mapping fails this test).

## Acceptance Criteria

- [ ] Markdown golden stable across runs (byte-identical given same artifact minus timestamp field, which comes from the artifact).
- [ ] Every REASONS literal has human text (completeness test).
- [ ] `explain` reproduces DESIGN.md §10.6#2 arithmetic legibly (snapshot).
- [ ] Invalid artifact never written (induced-invalid test asserts throw-before-write).
- [ ] Section caps + "and N more" behavior verified at 201 rows.

## Validation

`pnpm test src/report`; visual check of markdown rendered in a scratch GitHub gist/PR (paste screenshot in PR).

## Dependencies

02, 04, 21, 22 (REASONS), 11 (stats passthrough shape).

## Non-goals

SARIF (v2 §2.3), HTML reports, historical trend storage (stateless by ADR-004).

## Design References

DESIGN.md §16, §7 (purity, exit 2), §5.1 (artifact), §17 T4 (caps).
