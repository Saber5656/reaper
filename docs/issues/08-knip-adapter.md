# Title

knip adapter: TS/JS unused files, exports, types, class/enum members, dependencies

## Summary

Implement the knip adapter per DESIGN.md §9.3: pinned invocation via `npx --no-install`, JSON reporter parsing, issue-type→category mapping, golden-output tests.

## Context

knip is the sole TS/JS detector and the only v1 source of `unused-file` (the flagship auto-remove category). Its JSON shape is verified in `docs/research/dead-code-tools-survey.md`.

## Scope

In: `src/adapters/knip.ts` + golden fixtures `test/golden/knip/*.json`.
Out: removal (23/24), knip-config authoring for target repos (user's own), scoring.

## Detailed Requirements

1. Add `knip` as an exact-pinned runtime dependency of reaper (latest stable 5.x at implementation time; record the pin in the adapter as `TESTED_VERSION_RANGE = ">=5.55 <6"`, adjust to reality).
2. `probe`: resolution order — (a) target repo's `node_modules/.bin/knip` if its version satisfies the tested range (read its package.json), else (b) reaper's own bundled knip via `npx --no-install knip` resolved from reaper's install dir. Probe returns the chosen binary path + version; no network ever (`--no-install`, §17 T5).
3. `run` invocation (argv array, cwd = TS root): `knip --reporter json --no-exit-code --include files,exports,types,enumMembers,classMembers,dependencies,devDependencies` + `cfg.tools.knip.args` appended verbatim (array of strings from config; schema forbids non-string entries).
4. Parse stdout as JSON (§9.3 shape `{ issues: [ { file, owners?, <type>?: [...] } ] }`); ajv-lite structural check (hand-rolled guards acceptable); mapping:
   | knip key | category | symbolKind | symbol | notes |
   |---|---|---|---|---|
   | `files` (file-level entry) | unused-file | file | null | knip marks whole file unused |
   | `exports[]` | unused-export | export (or function/variable if knip provides) | `name` | `line`/`col` → startLine |
   | `types[]` | unused-export | type | `name` | interfaces/type aliases/enums |
   | `enumMembers[]` | unused-symbol | enum-member | `namespace.name` composite: symbol=`name`, prefix namespace into symbol as `Enum.member` | |
   | `classMembers[]` | unused-symbol | class-member | `Class.member` composite | |
   | `dependencies[]` / `devDependencies[]` | unused-dependency | dependency | package name | path = the declaring package.json (knip's `file`) |
   Ignored keys (`unlisted`, `unresolved`, `binaries`, `duplicates`, unknown future keys): skipped silently with a debug count.
5. `language`: `"ts"` when the file ends `.ts/.tsx/.mts/.cts`, else `"js"`; dependencies → language of the root (`ts`).
6. `detector`: `{ name: "knip", version: <probed>, nativeType: <knip key> }`.
7. Empty/absent issue arrays ⇒ zero findings, status ok. Non-JSON stdout ⇒ `E_ADAPTER_PARSE` (→ failed).
8. Golden tests: commit 3 real captured outputs (generated once during implementation from the issue-33 ts fixture and two synthetic cases: monorepo workspaces, deps-only) under `test/golden/knip/`; parse tests assert exact Finding lists (snapshot with fingerprints).
9. Conformance: `describeAdapterConformance(knipAdapter, …)` wired.

## Acceptance Criteria

- [ ] Golden parse tests pass; fingerprints stable across runs.
- [ ] Mapping table above fully covered by tests, including composite symbols (`Enum.member`) and the ignored-keys path.
- [ ] Probe prefers in-range repo-local knip; falls back to bundled; never fetches (fake exec asserts no install-ish argv).
- [ ] Conformance suite passes.
- [ ] Against fixture `fixtures/ts-app` (once 33 exists): produces the expected raw findings — tracked as an integration assertion in 33, noted here for traceability.

## Validation

`pnpm test src/adapters/knip*`; manual smoke: run the adapter against a scratch clone of a real OSS TS repo and eyeball the mapped categories (record the repo + command in the PR description).

## Dependencies

05, 06, 07.

## Non-goals

Driving knip `--fix` (never, ADR-002), authoring knip config into target repos, workspace re-implementation (§9.3).

## Design References

DESIGN.md §9.3, §9.2 matrix, §17 T5; research survey §2 (knip facts); ADR-002.
