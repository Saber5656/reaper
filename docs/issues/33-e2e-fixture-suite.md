# Title

E2E fixture suite: seeded fixture repos, trap corpus, precision hard gate, determinism gate

## Summary

Build the three fixture mini-repos (ts/py/go) with seeded dead code, alive-but-tricky traps, and keep-annotated items; plus the E2E harness that runs the real pipeline with real tools in CI and enforces the precision contract: zero traps proposed, ever (DESIGN.md §18.1).

## Context

ADR-003's promise is only as good as its enforcement. This suite is the enforcement: every confidence weight, heuristic rule, and policy predicate ultimately answers to these fixtures. It also collects the cross-issue integration assertions deferred from 08–12, 24–27, 31.

## Scope

In: `fixtures/{ts-app,py-app,go-app}` with `expected.json` each; `test/e2e/` harness; CI job installing real tools (uv, go, node) and running scan + propose `--dry-run`; determinism double-run gate (§18.3).
Out: live-GitHub publish testing (optional follow-up documented as a known unknown here), weight tuning itself.

## Detailed Requirements

1. Fixture content matrix — each fixture is a minimal but REAL project (installable deps, compiling code):
   - **ts-app** (pnpm, tsconfig, knip-clean config): dead: unused file, unused export (deletable), unused export with internal refs (demotion case), unused enum member, unused dependency (+ unused devDependency). Traps: file referenced only via dynamic `import(\`./pages/${name}\`)` pattern + string in a yaml; next-style `pages/` file; symbol in template literal; dep used only in a script body (`package.json scripts`); `.d.ts` export; keep-annotated export; generated `*.gen.ts` dead file (must be report-not-propose via S7+policy).
   - **py-app** (uv, pyproject): dead: unused function (vulture 60%), unused method, unused class, unused prod dependency. Traps: celery-decorated task, pytest fixture in conftest, `getattr(module, name)` registry target, function named in `routes.yml` (§10.6#2 reproduction), `__all__`-listed symbol with repoKind=library override in fixture config, keep-annotated function, `[project.scripts]` entry function.
   - **go-app** (go.mod with main): dead: unreachable function, unreachable method, unused direct require. Traps: `//go:linkname` function, `TestX` helper, reflect-using package function, build-tagged function, exported symbol under fixture config `repoKind: library`.
2. `fixtures/*/expected.json` schema (consumed by harness):
   ```jsonc
   { "items": [ { "match": { "category": "...", "path": "...", "symbol": "...?" },
                 "expect": "proposed" | "reported" | "suppressed" | "absent",
                 "minBand": "high|medium|low?", "mustNotPropose": true? , "note": "why this case exists" } ] }
   ```
   Every trap carries `mustNotPropose: true`. Every seeded-dead item documents its intended decision.
3. Harness (`test/e2e/run.test.ts`):
   - Per fixture: fresh copy to tmp, `git init` + commit (pipelines require git), tool bootstrap check (skip-with-loud-error if CI lacks a tool — CI must have all), run real `reaper scan` (built CLI), then `reaper propose --dry-run`.
   - Assertions: (a) **precision hard gate** — no `mustNotPropose` item has action `propose` (failure prints the finding's full explain output); (b) expected decisions/bands per item; (c) recall floor — every `expect: proposed|reported` item found (missing ⇒ fail listing adapter statuses); (d) removal validity — dry-run worktrees pass the language's builtin verify (tsc/compileall/go build) [deferred assertions from 24–27]; (e) `go-mod-deps` left fixture tree untouched [12]; (f) determinism — second scan byte-identical modulo `startedAt` (§18.3).
4. CI wiring: dedicated workflow job `e2e` (needs: unit job) on ubuntu; installs pinned uv/go/node versions; caches go modules + uv cache; runtime budget < 10 min (K4 measurement recorded in job summary).
5. Golden refresh workflow: `pnpm e2e:update-golden` regenerates adapter golden captures (08–12) from fixtures with a diff-review reminder printed.
6. Known unknown recorded: optional live-GitHub propose test against a scratch repo (manual `workflow_dispatch` job, `if: secrets.E2E_REPO_TOKEN` present) — specified but may land as follow-up.

## Acceptance Criteria

- [ ] All three fixtures + expected.json committed; every §18.1 trap class represented (checklist in PR maps trap→fixture item→defending signal/rule).
- [ ] E2E job green in CI with real tools; precision gate demonstrably fails when a trap's guard is disabled (one-off proof run documented in PR, then reverted).
- [ ] Recall floor + decision assertions green; determinism gate green.
- [ ] Runtime < 10 min; K4 latency measurement recorded.
- [ ] Deferred integration assertions from 08, 12, 24, 25, 26, 27, 31 all implemented here (traceability list in PR).

## Validation

CI `e2e` job on the issue's PR; local `pnpm test:e2e` documented in CONTRIBUTING stub.

## Dependencies

31 (pipelines), 32 not required; 08–12, 23–27, 29(dry-run path), 30.

## Non-goals

Performance benchmarking beyond the budget (15 has its own), fuzzing (v2), live publish test (optional follow-up).

## Design References

DESIGN.md §18 (entire), §10.6, §19 K4; ADR-003 (this suite enforces it).
