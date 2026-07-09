# Research: Dead-Code Detection Tool Survey (v1 detector selection)

Status: complete (informs DESIGN.md §9 and ADR-002)
Date: 2026-07-10
Method: vendor documentation review (knip.dev, pkg.go.dev, deptry.com, vulture README) verified via web on 2026-07-10. Exact pinned versions are resolved at implementation time; structural facts below (flags, output formats) were verified against current docs.

## 1. Why this survey exists

reaper v1 orchestrates existing per-language detectors instead of building its own reachability engine (ADR-002). That makes the detector inventory — exact invocations, output formats, native confidence semantics, and known blind spots — load-bearing design input. Every adapter issue (08–12) derives its parsing contract from this file and DESIGN.md §9.

## 2. Selected detectors

| Language | Tool | Categories produced | Output format | Native confidence | Executes repo code? |
|---|---|---|---|---|---|
| TS/JS | knip (v5.x) | unused files, exports, types, class/enum members, dependencies | `--reporter json`: `{ "issues": [ { "file", "owners", "<issueType>": [ { "name", "line", "col", "pos" } ] } ] }` | none | **Yes** — loads `knip.ts`/`knip.js` config and plugin resolution runs in-process |
| Python | vulture (v2.x) | unused functions, methods, classes, variables, attributes, properties | text lines: `path:line: unused function 'name' (60% confidence)` | 60–100% per finding | No (pure AST) |
| Python | deptry (v0.2x) | unused (obsolete) dependencies = rule `DEP002` | `--json-output <file>`: `[{ "error": { "code": "DEP002", "message": "..." }, "module": "...", "location": { "file": "pyproject.toml", "line": null, "column": null } }]` | none | No |
| Go | `golang.org/x/tools/cmd/deadcode` | unreachable functions (RTA call graph from `main` roots) | `-json`: `[ { "Name", "Path", "Funcs": [ { "Name", "Position": { "File", "Line", "Col" }, "Generated", "Marker" } ] } ]` | none (but `Generated` flag) | Compiles packages (build-tag/codegen aware), does not execute |
| Go | `go mod tidy` diff | unused module requirements | diff of `go.mod` before/after tidy in a scratch copy | n/a (exact) | No |

Key per-tool facts and caveats:

### knip
- Nine built-in reporters; we consume `json`. Each per-file object carries one key per enabled issue type; empty arrays present.
- Issue types used by reaper: `files`, `exports`, `types`, `enumMembers`, `classMembers`, `dependencies`, `devDependencies`. Ignored in v1: `unlisted`, `unresolved`, `binaries`, `duplicates` (they are hygiene issues, not dead code).
- knip understands JS/TS monorepo workspaces natively — reaper delegates JS workspace traversal to knip rather than re-implementing it.
- knip has `--fix`, which we deliberately do NOT use: it removes code without a confidence/safety layer. reaper owns removal (DESIGN.md §12).
- Security-relevant: a `knip.ts` config file is executed. Running reaper on a ref therefore executes that ref's code. This shapes the trust model (ADR-005).
- ts-prune is in maintenance mode and its README recommends knip; it is excluded.

### vulture
- Confidence: 100% only for code provably unreachable within analyzed files; 60–90% are heuristic per-construct estimates. reaper maps these onto its own base-score scale (DESIGN.md §10.4) rather than passing them through.
- Whitelist convention: a Python file of "used" names passed as an extra path; `--make-whitelist` generates one. reaper maps its own suppression model onto this (adapter passes ignore names via CLI, not by writing whitelist files into the repo).
- Exit code varies by failure class and version (findings vs. usage errors). The adapter must parse stdout and must NOT infer success/failure from exit code alone; treat "parse succeeded" as authoritative.
- Blind spots: `getattr`/`globals()`/metaprogramming, framework-invoked callables (Django views, celery tasks, pytest fixtures). These drive the Python entry-point heuristics (issue 18) and dynamic-usage scanner (issue 16).

### deptry
- Supports PEP 621 `pyproject.toml`, Poetry, uv, and requirements files; configured via `[tool.deptry]` or CLI flags.
- DEP002 is not evaluated for development dependency groups by design — reaper therefore only ever proposes removal of *production* Python dependencies.
- Rule codes other than DEP002 (DEP001 missing, DEP003 transitive, DEP004 misplaced dev) are out of scope for v1 (not dead code).

### deadcode (golang.org/x/tools)
- Roots are `main` packages only; libraries without a `main` in the module produce no useful signal — the Go adapter must detect this and mark the whole language pass `skipped(no-main-roots)` instead of emitting nothing silently.
- `-test` includes test binaries as roots: with it, test-only helpers count as live; without it, anything reachable only from tests is dead. reaper runs WITH `-test` by default (conservative; avoids proposing code that tests still exercise) and records `testOnlyUsage` by diffing a second run without `-test` (DESIGN.md §9.6).
- `Generated: true` findings map to the `generated.file` penalty signal.
- Reflection: RTA models some reflective edges but `reflect`-heavy code can still be misreported — the Go dynamic-usage scanner (issue 16) penalizes packages importing `reflect`/`unsafe`/`plugin`.
- Invocation is pinned: `go run golang.org/x/tools/cmd/deadcode@<pinned> -json -test ./...`.

## 3. Tools considered and rejected

| Tool | Reason rejected |
|---|---|
| ESLint `no-unused-vars`, `tsc noUnusedLocals` | In-file scope only; already solved and auto-fixed by linters. reaper's floor is project-level reachability (README positioning). Overlap would add noise, not value. |
| ts-prune | Maintenance mode; superseded by knip. |
| depcheck (JS) | knip already reports `dependencies`/`devDependencies`; a second JS dep detector adds conflict surface, not precision. Revisit only as a corroboration source (v2). |
| unimport / autoflake (Python) | In-file unused imports — linter territory (ruff F401), non-goal. |
| staticcheck U1000 (Go) | Overlaps deadcode with a different engine; deadcode's whole-program RTA is the stronger primary. Candidate v2 corroboration source. |
| Coverage-based detection (run tests, mark uncovered) | Requires executing the project and conflates "untested" with "dead"; unacceptable false-positive profile for auto-PR. Rejected for all versions unless used as a *negative* signal (v2 idea). |
| SCIP/LSIF own-graph engine | Highest ceiling, highest cost; deferred — see ADR-002. |

## 4. Cross-tool observations that shaped the design

1. **No tool crosses languages.** Nothing merges TS+Python+Go findings into one stream — that gap is reaper's integration value.
2. **No tool owns the act-on-it workflow.** knip `--fix` deletes locally with no confidence gate; nothing opens reviewable PRs with provenance. That gap is reaper's product value.
3. **Native confidence is absent or crude.** Only vulture emits one, and it is per-construct heuristic. A calibrated cross-tool confidence model must be reaper's own (DESIGN.md §10).
4. **Every tool has dynamic-usage blind spots.** Each ecosystem fails the same way: string-keyed registries, reflection, framework entry points. A language-agnostic *string-reference verifier* (grep the identifier across the whole repo, including non-code files) is cheap and catches a large share of these; it is reaper's primary independent safety signal (issue 15).
5. **Output formats are stable but versioned.** All adapters pin tool versions and carry golden-fixture parse tests so an upstream format change breaks CI, not users (issues 08–12).
