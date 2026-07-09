# Title

go-mod-deps adapter: unused Go module requirements via tidy-diff in a scratch overlay

## Summary

Implement the adapter that detects unused direct `go.mod` requirements by running `go mod tidy` against a scratch copy of the module manifests and diffing the `require` sets, per DESIGN.md §9.7 — without ever touching the working tree.

## Context

`go mod tidy` computes the exact requirement set. Diffing it against the committed `go.mod` yields zero-false-positive unused-dependency findings (base 0.90) — the strongest signal in the whole system.

## Scope

In: `src/adapters/go-mod-deps.ts` + go.mod parsing helpers + tests.
Out: applying tidy as removal (26 delegates to the same helper, exported), deadcode (11).

## Detailed Requirements

1. Scratch overlay procedure (§9.7), per Go root:
   a. `mkdtemp` scratch dir; copy `go.mod` + `go.sum` (if present) into it.
   b. Create `<scratch>/tidy-probe/` containing ONLY those copies — but `go mod tidy` needs the source to compute imports, so instead: run `go mod tidy` with the real module as source but redirected manifests — implementation: copy the ENTIRE module source? Too heavy. Correct mechanism (frozen here): run `go list -deps -f '{{if not .Standard}}{{.Module.Path}}{{end}}' ./...` in the real root (read-only) to compute the used module set; parse `go.mod` `require` directives (direct, non-`// indirect`) with a hand-rolled line parser; unused = direct requires ∖ used set ∖ tool directives.
   c. Cross-check (belt and braces): also run `go mod why -m <candidate>` for each candidate (capped at 20 candidates); `(main module does not need module …)` confirms; anything else drops the candidate with debug log.
2. Parsing `go.mod`: support single-line `require x vY` and block `require ( … )`; track `// indirect` comments; ignore `replace`/`exclude`/`tool` lines except: modules referenced by `tool` directives are never candidates.
3. Mapping per confirmed unused module: `category: "unused-dependency"`, `language: "go"`, `symbol: <module path>`, `symbolKind: "dependency"`, `location.path`: the `go.mod` repo-relative, `startLine`: the require line, `nativeType: "tidy-diff"`, detector name `go-mod-deps`, version = `go version` string.
4. Read-only guarantee: no command in this adapter may mutate the working tree — enforced by running `git status --porcelain` before/after in the adapter's integration test (kit already asserts no-writes; add explicit test here because `go list` can update caches — caches live outside the repo via env `GOFLAGS=-mod=readonly` set on every invocation).
5. `probe`: go present; `go.mod` parseable. Network: `go list`/`go mod why` may need module downloads — set `GOFLAGS=-mod=readonly` and tolerate failures by dropping to status `failed` with hint about running `go mod download` first (documented in doctor hints).
6. Tests: go.mod parser table tests (blocks, indirect, tool directives, malformed ⇒ `E_ADAPTER_PARSE`); fake-exec pipeline test (list output + why outputs → findings); readonly-flag assertion on every go invocation.
7. Conformance suite wired.

## Acceptance Criteria

- [ ] Direct-unused module in the go fixture is found; indirect and tool-directive modules never are (tests).
- [ ] `go mod why` disagreement drops the candidate (test).
- [ ] Every spawned go command carries `GOFLAGS=-mod=readonly` (fake-exec assertion).
- [ ] Working tree untouched after a real run (integration test on fixture once 33 lands; noted there).
- [ ] Conformance suite passes.

## Validation

`pnpm test src/adapters/go-mod-deps*`; manual smoke on a real Go repo with a known-unused require (add one to a scratch clone; record in PR).

## Dependencies

05, 06, 07.

## Non-goals

Running tidy as detection in-tree (never), go workspaces `go.work` multi-module resolution beyond per-root iteration (K-class unknown; document if hit), vendored modules (`vendor/` skipped by discovery).

## Design References

DESIGN.md §9.7, §10.3 (base 0.90), §12.4 (removal reuses helper), §17 T3; ADR-002.
