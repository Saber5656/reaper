# Title

vulture adapter: Python unused symbols with native-confidence capture

## Summary

Implement the vulture adapter per DESIGN.md §9.4: pinned `uvx` invocation with python3 fallback, line-format parsing (discarding unused-import lines), whitelist passthrough, golden tests.

## Context

vulture is the sole Python symbol detector. Its output is plain text with a native 60–100% confidence that reaper maps into base scores (§10.4) — captured here, mapped in issue 21.

## Scope

In: `src/adapters/vulture.ts` + golden fixtures `test/golden/vulture/*.txt`.
Out: python removal (25), scoring/mapping (21), suppression semantics (20).

## Detailed Requirements

1. Pin: `VULTURE_PIN = "2.14"` (adjust to latest stable at implementation; exact `==` pin).
2. `probe`: (a) `uv --version` ok ⇒ runner = `uvx vulture==<pin> --version`; (b) else `python3 -c "import vulture, sys; print(vulture.__version__ if hasattr(vulture,'__version__') else 'unknown')"` — importable ⇒ runner = `python3 -m vulture`; version outside tested range ⇒ available with warn hint. Neither ⇒ unavailable, hint "install uv or pip install vulture".
3. `run` invocation (cwd = python root): `<runner> . --min-confidence <cfg.tools.vulture.minConfidence>` plus whitelist passthrough: if `whitelist.py` or `.vulture_whitelist.py` exists at the root, append as extra path args (§9.4).
4. Parse stdout with the exact anchored regex from §9.4:
   `^(?<path>.+?):(?<line>\d+): unused (?<kind>function|method|class|variable|attribute|property|import) '(?<name>[^']+)' \((?<conf>\d+)% confidence\)$`
   - `kind === "import"` ⇒ line discarded (linter territory §1.1), counted in a debug stat.
   - kind mapping → symbolKind: function→function, method→method, class→class, variable→variable, attribute/property→variable (attribute nuance recorded in `nativeType`).
   - `nativeType`: `unused-<kind>`; `nativeConfidence`: int 60–100.
   - Non-matching, non-empty stdout lines: tolerated up to 5% of lines (vulture may print notes); beyond that ⇒ `E_ADAPTER_PARSE`.
5. Exit-code policy (§9.4 + research survey): do NOT branch on exit code for success; success = stdout fully parsed AND stderr contains no `Traceback`/`error:` marker. A `Traceback` in stderr ⇒ `E_ADAPTER_PARSE` with stderr head.
6. All findings: `category: "unused-symbol"`, `language: "python"`.
7. Golden tests: 2 committed real outputs (from the issue-33 py fixture + a synthetic with all kinds incl. an import line and a 100% confidence entry); assert import-discard, confidence capture, tolerance rule (craft a file with >5% garbage ⇒ parse error).
8. Conformance suite wired.

## Acceptance Criteria

- [ ] Golden tests pass; `unused import` never yields a Finding.
- [ ] nativeConfidence values (60/90/100) present verbatim on findings.
- [ ] uv-absent fallback path covered by fake-exec test; neither-available ⇒ skipped with hint.
- [ ] Traceback-in-stderr ⇒ adapter failed status, reason includes head of stderr.
- [ ] Conformance suite passes.

## Validation

`pnpm test src/adapters/vulture*`; manual smoke against a scratch real Python repo (record in PR).

## Dependencies

05, 06, 07.

## Non-goals

Writing whitelist files into target repos (never); mapping confidence→base (21); dynamic-usage compensation (16/18).

## Design References

DESIGN.md §9.4, §10.4; research survey §2 (vulture facts, exit-code caveat); ADR-002.
