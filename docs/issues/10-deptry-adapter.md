# Title

deptry adapter: Python unused (DEP002) production dependencies

## Summary

Implement the deptry adapter per DESIGN.md §9.5: pinned `uvx` invocation, `--json-output` to a scratch file, DEP002-only filtering, golden tests.

## Context

deptry provides Python `unused-dependency` findings — one of the two default auto-propose categories. It only evaluates DEP002 for production dependencies, which is exactly the conservative envelope we want (research survey §2).

## Scope

In: `src/adapters/deptry.ts` + golden fixtures `test/golden/deptry/*.json`.
Out: dependency removal (26), scoring.

## Detailed Requirements

1. Pin: `DEPTRY_PIN = "0.23.0"`-style exact pin (latest stable at implementation).
2. `probe`: requires `uv` (`uvx deptry==<pin> --version`); no pip fallback in v1 (deptry is uv-installable everywhere we support; document hint "install uv"). Unavailable ⇒ skipped with hint.
3. `run` (cwd = python root): create scratch file via `mkdtemp` under os tmpdir (NEVER inside the repo, §9.5); invoke `uvx deptry==<pin> . --json-output <scratch>/deptry.json`; deptry's nonzero exit on findings is expected — success = scratch JSON exists and parses as an array.
4. Parse: array of `{ error: { code, message }, module, location: { file, line, column } }`.
   - Keep only `error.code === "DEP002"`; other codes counted into a debug stat and dropped (§9.5).
   - Mapping: `category: "unused-dependency"`, `language: "python"`, `symbol: module`, `symbolKind: "dependency"`, `location.path`: `location.file` normalized repo-relative (typically `pyproject.toml`), `startLine`: `location.line` (nullable), `nativeType: "DEP002"`.
5. Config passthrough: none beyond pin in v1 (`tools.deptry` reserved); target-repo `[tool.deptry]` settings are respected implicitly by deptry itself.
6. Scratch hygiene: temp dir removed in `finally`; JSON read capped at 8 MiB.
7. Golden tests: 2 committed outputs (issue-33 py fixture capture + synthetic containing DEP001/DEP003/DEP004 noise to prove filtering); malformed JSON ⇒ `E_ADAPTER_PARSE`.
8. Conformance suite wired.

## Acceptance Criteria

- [ ] Only DEP002 entries become findings; other rule codes filtered (test).
- [ ] Findings point at the dependency-declaring file with nullable line preserved.
- [ ] Temp dir cleaned up on success and on parse failure (test with fake fs or tmpdir listing).
- [ ] uv missing ⇒ status skipped with actionable hint; `--strict-tools` behavior verified via registry test (07).
- [ ] Conformance suite passes.

## Validation

`pnpm test src/adapters/deptry*`; manual smoke on a real uv-managed project (record in PR).

## Dependencies

05, 06, 07.

## Non-goals

DEP001/003/004 handling (not dead code, §9.5), requirements.txt-only legacy layouts beyond what deptry itself supports, dev-group deps (excluded by deptry by design).

## Design References

DESIGN.md §9.5, §9.2; research survey §2 (deptry facts); ADR-002.
