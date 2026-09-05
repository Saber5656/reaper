# Title

String-reference verifier: repo-wide identifier search producing S1/S2/S3/S12

## Summary

Implement reaper's primary independent safety signal: for every candidate symbol, search the entire indexed repo for the identifier and classify hits into non-code (S1), code-string (S2), comment-only (S3), and test-only usage (S12), per DESIGN.md §10.5.

## Context

Every detector's blind spot is dynamic/string-keyed usage (research survey §4). A word-boundary search across ALL files — including templates, YAML, SQL — is detector-independent and catches the false positives that would otherwise become catastrophic PRs (worked example §10.6#2). This collector is the single most precision-critical component after the policy engine.

## Scope

In: `src/signals/string-reference.ts` + tests. Uses the RepoFileIndex from 14.
Out: dynamic-pattern detection in code semantics (16), performance indexing beyond the batching below (K6 v2).

## Detailed Requirements

1. Eligible findings: categories `unused-export`, `unused-symbol`, and `unused-dependency` (module name hits in non-code config files matter: e.g. a dep referenced in `serverless.yml` plugins). `unused-file` findings use the file's basename-without-extension as the searched identifier (import-by-string catch) — plus, for TS files, any `export default`-adjacent name is out of scope (keep simple: basename only).
2. Identifier extraction: `location.symbol`; for composite symbols (`Enum.member`, `(*Client).Close`) search the member name (`member`, `Close`) — the more specific token; symbols shorter than 3 chars or in the stop list (`main, init, run, get, set, id, new, test`) produce a `string-reference.skipped-short` zero-delta signal instead of S1–S3 (too noisy to search; policy: skip means no propose-block, but also no exoneration).
3. Search semantics:
   - Word-boundary regex `\b<escaped>\b` case-sensitive, over every indexed file except: the finding's own file, lockfiles (`pnpm-lock.yaml, package-lock.json, yarn.lock, go.sum, uv.lock, poetry.lock`), and `.reaper/`.
   - Implementation: batched single-pass scan — compile one combined regex per ≤200 identifiers (alternation with named groups is overkill: use Aho-Corasick-style manual scan or per-file `indexOf` prefilter then per-identifier boundary regex). Complexity budget: one full file read pass per scan regardless of finding count (acceptance criterion below).
4. Hit classification per identifier (first matching rule per file, aggregate across files):
   - file class `non-code` ⇒ S1 `string-reference.non-code-hit` (−0.40).
   - file class `code` and the hit lies inside a string literal ⇒ S2 `code-string-hit` (−0.30). String-literal detection: lightweight lexer for quotes/backticks/py-triple-quotes per language family — heuristic; documented accuracy tradeoff: false S2 positives are safe (they only lower confidence).
   - hit inside a comment (`// # /* */ docstrings`) and no other hit class fired anywhere ⇒ S3 `comment-only-hit` (−0.05).
   - hits ONLY in `test`-class code files (normal code hits there, not strings) ⇒ S12 `test-only.usage` (delta 0, `meta.testOnlyUsage=true`).
   - plain code-identifier hits in non-test files: NO signal (the detector already claims unreachability; a bare textual hit in code is usually the declaration's siblings — but see cap note) — however if > 0 such hits exist in files OTHER than the declaration file, emit zero-delta informational `string-reference.code-hit` with evidence (helps `explain` debugging).
   - Evidence: up to 5 `path:line` entries per signal; total hit processing per identifier capped at 100 hits (beyond: stop, keep strongest class — S1 outranks S2 outranks S3).
5. Strongest-class-only: at most ONE of S1/S2/S3 per finding (the strongest present); S12 independent.
6. Determinism: evidence sorted by `(path, line)`.
7. Tests: classification matrix over a synthetic tree (yaml hit, template hit, py f-string, ts template literal, comment-only, test-only, stop-list, short symbol, composite symbol member extraction, 100-hit cap, lockfile exclusion, own-file exclusion); batching correctness (3 identifiers, one pass — assert via instrumented index read counts); determinism snapshot.

## Acceptance Criteria

- [ ] Full classification matrix green, including the §10.6#2 reproduction (vulture finding + routes.yml hit ⇒ S1).
- [ ] One index read pass per file per scan (instrumentation assertion), independent of finding count.
- [ ] At most one of S1/S2/S3 per finding; S12 sets meta and never a delta.
- [ ] Stop-list and short identifiers yield `skipped-short` marker signal.
- [ ] 5k findings × 2k files synthetic benchmark completes < 30 s on CI (guard for K6; skip-on-CI-flaky allowed with local proof recorded in PR).

## Validation

`pnpm test src/signals/string-reference*`; benchmark run output pasted into the PR.

## Dependencies

14 (index + contract).

## Non-goals

Semantic import resolution (detectors' job), fuzzy/case-insensitive matching, persisted index (K6 v2).

## Design References

DESIGN.md §10.5 S1–S3/S12, §10.6 examples, §18.1 traps; ADR-003 (second-opinion principle); research survey §4.
