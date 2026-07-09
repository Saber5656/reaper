# Title

go-deadcode adapter: Go unreachable functions via x/tools deadcode (report-only)

## Summary

Implement the Go deadcode adapter per DESIGN.md §9.6: pinned `go run` invocation with `-json -test`, no-main-roots skip, dual-run testOnly accounting, Generated flag capture, golden tests.

## Context

`golang.org/x/tools/cmd/deadcode` performs whole-program RTA from `main` roots — precise but blind to reflection and useless without main packages. Go symbol findings are report-only in v1 (capability matrix §9.2); their value is the unified report.

## Scope

In: `src/adapters/go-deadcode.ts` + golden fixtures `test/golden/go-deadcode/*.json`.
Out: any Go removal (v2), go.mod deps (12).

## Detailed Requirements

1. Pin: `DEADCODE_PIN = "v0.<latest>"` exact module version at implementation time; invocation `go run golang.org/x/tools/cmd/deadcode@<pin> -json -test ./...` (cwd = go root). First-run module download is expected and documented (§9.6, §17 T5 exception: exact-pinned `go run` fetch).
2. `probe`: `go version` parses (≥ go1.22 tested floor); go missing ⇒ skipped with hint. Probe also runs `go list -f '{{.Name}}' ./...` capped output to detect `main` packages; none ⇒ adapter returns status `skipped`, reason `no-main-roots` (§9.6) — this check runs in probe so scan reports it before any expensive analysis.
3. `run` (primary): parse `-json` output — array of `{ Name, Path, Funcs: [ { Name, Position: { File, Line, Col }, Generated } ] }` (research survey §2). Mapping per func:
   - `category: "unused-symbol"`, `language: "go"`.
   - `symbolKind`: `method` if `Name` matches `/^\(?\*?[A-Za-z_][A-Za-z0-9_]*\)?\.[A-Za-z_]/` receiver form or contains a dot after a type, else `function`. `symbol`: the `Name` as printed (keep receiver qualification, e.g. `(*Client).Close`).
   - `location.path`: `Position.File` normalized repo-relative (deadcode may print absolute paths — relativize; outside-repo paths (GOROOT/std) are dropped with debug count).
   - `startLine`: `Position.Line`. `nativeType: "deadfunc"`.
   - `Generated: true` ⇒ stash on the RawFinding via `detector.nativeType = "deadfunc-generated"` (signal S7 consumer reads this marker; issue 14 collector).
4. `run` (secondary, testOnly accounting §9.6): second invocation WITHOUT `-test`; compute set difference (findings present only without `-test` are live-only-via-tests). v1: these are NOT emitted; return them via adapter status metadata `reason: "testOnlyDead=<n>"`? — No: extend `AdapterRunStatus` with optional `stats?: Record<string, number>` (small contract addition, update issue 07 kit) and set `stats.testOnlyDead`. Report renderer (30) surfaces it.
5. Timeout: this adapter overrides default to 600 s (whole-program analysis on large modules).
6. Golden tests: 2 committed JSON outputs (issue-33 go fixture capture + synthetic with methods, Generated entries, absolute paths); dual-run difference test with fake exec returning different sets.
7. Conformance suite wired.

## Acceptance Criteria

- [ ] Mapping covers functions, receiver methods, Generated flag, absolute-path relativization, std-lib drop.
- [ ] Module without main packages ⇒ probe yields skipped `no-main-roots`; nothing runs.
- [ ] Dual-run difference produces `stats.testOnlyDead` and suppresses those findings (test).
- [ ] Conformance suite passes; contract addition (`stats`) reflected in 07's types + kit.

## Validation

`pnpm test src/adapters/go-deadcode*`; manual smoke on a real Go CLI repo (record in PR).

## Dependencies

05, 06, 07.

## Non-goals

Go symbol removal (v2, §2.2/§2.3), staticcheck corroboration (v2), build-tag matrix coverage (single default build config in v1 — known limitation documented in report footnotes).

## Design References

DESIGN.md §9.6, §9.2, §10.5 S7; research survey §2 (deadcode facts); ADR-002.
