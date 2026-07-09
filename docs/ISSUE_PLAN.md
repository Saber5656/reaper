# reaper — v1 Issue Plan

Status: authoritative execution plan derived from [DESIGN.md](./DESIGN.md). GitHub Issues are generated from `docs/issues/*.md`; when they drift, these files win.

## 1. v1 completion statement

**v1 is complete when all 35 issues below are closed and validated.** At that point reaper:

- runs as a CLI (`scan`, `propose`, `report`, `init`, `doctor`, `explain`) and as a GitHub Action (report + propose modes) — DESIGN.md §7/§15;
- detects project-level dead code in TS/JS (knip), Python (vulture + deptry), and Go (deadcode + go-mod tidy-diff) per the capability matrix §9.2;
- scores every finding with the deterministic explainable confidence model §10 and decides propose/report/suppress per §11;
- automatically opens budgeted, deduplicated, provenance-carrying PRs for high-confidence removable findings (unused files, unused dependencies by default; exports/symbols opt-in), removal passing the verify gate §13;
- honors user sovereignty (keep-annotations, protect globs, close-PR-to-dismiss) and the ADR-005 security posture;
- proves its precision contract in CI: zero alive traps proposed across the fixture corpus (§18.1);
- ships with user docs, SECURITY.md, provenance-enabled release pipeline, and hardening controls.

No v1 product behavior exists outside this plan except newly discovered implementation unknowns (§8 below); §2.2/§2.3 of DESIGN.md list what is deliberately absent.

## 2. Issue list in recommended execution order

| # | Issue file | Title (short) | Size | Wave |
|---|---|---|---|---|
| 01 | issues/01-project-bootstrap.md | Toolchain + CI scaffold | M | W0 |
| 02 | issues/02-finding-model-and-fingerprint.md | Finding model + fingerprint | M | W0 |
| 03 | issues/03-config-schema-and-loader.md | Config loader + safety floors | M | W0 |
| 04 | issues/04-cli-skeleton.md | CLI skeleton + init | M | W0 |
| 05 | issues/05-subprocess-runner.md | Hardened ExecRunner | M | W0 |
| 06 | issues/06-workspace-discovery.md | Workspace discovery | M | W0 |
| 07 | issues/07-adapter-contract-and-registry.md | Adapter contract + kit | M | W1 |
| 08 | issues/08-knip-adapter.md | knip adapter | M | W1 |
| 09 | issues/09-vulture-adapter.md | vulture adapter | S | W1 |
| 10 | issues/10-deptry-adapter.md | deptry adapter | S | W1 |
| 11 | issues/11-go-deadcode-adapter.md | go-deadcode adapter | M | W1 |
| 12 | issues/12-go-mod-deps-adapter.md | go-mod-deps adapter | M | W1 |
| 13 | issues/13-doctor-command.md | doctor command | S | W1 |
| 14 | issues/14-signal-framework.md | Signal framework + S7/S11 | M | W2 |
| 15 | issues/15-string-reference-verifier.md | String-reference verifier | L | W2 |
| 16 | issues/16-dynamic-usage-scanner.md | Dynamic-usage scanner | M | W2 |
| 17 | issues/17-entrypoint-heuristics-ts.md | TS entrypoint pack | M | W2 |
| 18 | issues/18-entrypoint-heuristics-python.md | Python entrypoint pack | M | W2 |
| 19 | issues/19-entrypoint-heuristics-go.md | Go entrypoint pack | S | W2 |
| 20 | issues/20-suppression-and-keep-annotations.md | Suppression collector | M | W2 |
| 21 | issues/21-confidence-scoring-engine.md | Scoring engine | M | W2 |
| 22 | issues/22-decision-policy-engine.md | Policy engine | M | W2 |
| 23 | issues/23-removal-engine-and-file-deletion.md | Removal core + file delete | M | W3 |
| 24 | issues/24-ts-symbol-removal.md | TS symbol removal | L | W3 |
| 25 | issues/25-python-symbol-removal.md | Python symbol removal | L | W3 |
| 26 | issues/26-dependency-removal.md | Dependency removal | L | W3 |
| 27 | issues/27-verify-gate.md | Verify gate | M | W3 |
| 28 | issues/28-git-worktree-layer.md | Git worktree layer | M | W3 |
| 29 | issues/29-github-pr-publisher.md | PR publisher + dedupe | L | W4 |
| 30 | issues/30-report-renderers.md | Report renderers + explain | M | W4 |
| 31 | issues/31-scan-and-propose-pipelines.md | Pipeline integration | L | W4 |
| 32 | issues/32-github-action.md | GitHub Action | M | W4 |
| 33 | issues/33-e2e-fixture-suite.md | E2E fixtures + precision gate | L | W5 |
| 34 | issues/34-user-docs-and-security-policy.md | User docs + SECURITY.md | M | W5 |
| 35 | issues/35-release-pipeline-and-hardening.md | Release + hardening | M | W5 |

Size: S ≈ half day, M ≈ 1 day, L ≈ 2 days for a focused implementation agent.

## 3. Dependency table

| Issue | Hard dependencies | Soft/coordination |
|---|---|---|
| 01 | — | — |
| 02 | 01 | 22 amends Decision type (coordinated) |
| 03 | 01, 02 | 21 exports HIGH_FLOOR (refactor) |
| 04 | 01, 02, 03 | bodies filled by 13/30/31 |
| 05 | 01, 02 | — |
| 06 | 01, 02, 03, 05 | — |
| 07 | 02, 03, 05, 06 | 21 removes temp base table; 11 adds `stats` field |
| 08 | 05, 06, 07 | golden capture from 33 fixtures |
| 09 | 05, 06, 07 | same |
| 10 | 05, 06, 07 | same |
| 11 | 05, 06, 07 | 14 (generated marker), 19 (symbol parse helper) |
| 12 | 05, 06, 07 | 26 reuses tidy helper |
| 13 | 04, 06, 07 | renders 08–12 probes as they land |
| 14 | 02, 03, 05, 06, 07 | 11 marker convention |
| 15 | 14 | extracts lex.ts consumed by 16/18/20 |
| 16 | 14, 15 | — |
| 17 | 14, 06 | pack contract for 18/19; shared config-glob helper |
| 18 | 14, 17, 15/16 (lex) | creates toml-lite used by 26 |
| 19 | 14, 17, 11 | — |
| 20 | 14, 15/16 (lex) | conventions mirrored in 29 PR body |
| 21 | 02, 07, 14 | refactors 03/07 constants |
| 22 | 02, 03, 21 | REMOVABLE consumed by 23; dismissed set from 29 |
| 23 | 02, 03, 22 | hostile fixtures reused by 24–26 |
| 24 | 23, 08 | — |
| 25 | 23, 09 | pins file shared with 08–12 |
| 26 | 23, 18 (toml-lite), 06, 08/10/12 | — |
| 27 | 05, 03, 23–26 (changedFiles contract) | verified flag consumed by 29 |
| 28 | 05, 02 | identity override used by 32 |
| 29 | 02, 03, 05, 22, 27, 28 | — |
| 30 | 02, 04, 21, 22 | 11 stats shape |
| 31 | 06, 07, 14, 21, 22, 30 (min); 23–29 (full) | replaces 04 stubs |
| 32 | 30, 31, 28, 05 | dist gate enforced by 35 |
| 33 | 31, 08–12, 23–27, 29, 30 | absorbs deferred integration assertions |
| 34 | 03, 04, 32, 33 (material) | draftable after 22 |
| 35 | 01, 32, 33, 34 | human gate: package name (K1) |

Critical path: 01 → 02 → 03/05 → 06 → 07 → 14 → 21 → 22 → 23 → 27 → 29 → 31 → 33 → 35.
Parallelizable clusters: {08,09,10,11,12} after 07; {15,16,17,18,19,20} after 14 (with the lex.ts ordering: 15 → 16/18/20); {24,25,26} after 23; {30,32} alongside 29/31.

## 4. Implementation waves

| Wave | Issues | Exit criterion (wave gate) |
|---|---|---|
| W0 Foundations | 01–06 | CLI builds; config+discovery+runner unit-tested; CI green |
| W1 Detection | 07–13 | `doctor` shows all 5 adapters; golden parse tests green; raw findings normalize with fingerprints |
| W2 Confidence | 14–22 | DESIGN §10.6 worked examples reproduced end-to-end on synthetic inputs; policy decision table 100% branch-covered |
| W3 Removal | 23–28 | All strategies + verify gate green incl. hostile-path suite; real-git worktree lifecycle tests green |
| W4 Publication | 29–32 | `scan`/`propose --dry-run` run end-to-end; Action guard matrix green; report/explain golden |
| W5 Assurance & release | 33–35 | Precision hard gate green in CI with real tools; docs complete; `v0.1.0-rc` pipeline run |

Wave gates are merge-discipline checkpoints, not calendar phases; issues within a wave parallelize per §3.

## 5. Coverage table (DESIGN.md § → issues)

| DESIGN.md section | Covered by |
|---|---|
| §1 Positioning | 34 (README) |
| §2 Goals/non-goals | plan-wide; §2.3 guarded by Non-goals sections in every issue |
| §3 Architecture/stack | 01 (stack), 31 (dataflow) |
| §4 Layout | 01; 23 amends (path-safety.ts) |
| §5 Data model | 02 |
| §6 Configuration | 03; 34 (reference docs) |
| §7 CLI contract | 04; 13/30/31 (bodies) |
| §8 Discovery | 06 |
| §9.1 Adapter contract | 07 |
| §9.2 Capability matrix | 22 (REMOVABLE data), 34 (docs) |
| §9.3–§9.7 Adapters | 08, 09, 10, 11, 12 |
| §10.1–§10.4 Scoring | 21 |
| §10.5 Signals S1–S3/S12 | 15 · S4/S5: 16 · S6/S8: 17/18/19 · S7/S11: 14 · S9: 20 · S10: 21 |
| §10.6 Worked examples | 21 (unit), 33 (fixtures) |
| §11 Policy | 22 |
| §12.1/§12.2 Removal core | 23 |
| §12.3 Symbol removal | 24 (TS), 25 (Python) |
| §12.4 Dependency removal | 26 |
| §13 Verify gate | 27 |
| §14.1 Git | 28 |
| §14.2–§14.4 Publication | 29 |
| §15 Action | 32 |
| §16 Reporting | 30 |
| §17 Security model | cross-cutting: 05 (T2/T4), 23 (T3), 03 (T6), 29 (T7/T8), 32 (T1), 35 (T5/T9), 34 (docs) |
| §18 Quality | 33 (+ per-issue Validation sections) |
| §19 Known unknowns | §8 below |
| §20 Glossary | n/a (reference) |

Every DESIGN section with implementable behavior maps to ≥1 issue; no issue implements behavior absent from DESIGN.md.

## 6. Validation strategy (whole product)

1. **Per-issue gates:** every issue's Validation section runs in CI on its PR (unit + golden + integration layers per DESIGN §18.2).
2. **Precision hard gate (the product contract):** 33's trap corpus — zero `mustNotPropose` items proposed, enforced on every PR from W5 onward; a bypass is classified a security vulnerability (34's SECURITY.md).
3. **Recall floor:** every seeded-dead fixture item detected and correctly banded (33).
4. **Determinism gate:** double-scan byte-identity (§18.3) in CI.
5. **Security invariants as tests:** env-allowlist and token-reach (05), path hostility (23), Action guard matrix (32), `pull_request_target` grep gate + dist-drift + SHA-pin audit (35).
6. **Dogfooding:** weekly report-mode run of reaper on this repository (35) — the living demo and drift alarm.
7. **Human gates preserved:** package name (35/K1), any weight change requires a fixture case (ADR-003, CONTRIBUTING rule in 34), merges of reaper PRs in target repos are always human (product invariant).

## 7. Deferred v2 items (from DESIGN §2.3 — do not implement in v1)

Own import-graph engine (Python unused files, cross-language consistency); Go symbol removal (`go/ast` rewriting); corroboration detectors (staticcheck, depcheck); SARIF; GitLab driver; per-finding bisect on verify failure; auto-rebase of open reaper PRs; coverage-as-negative-signal; LLM re-ranking atop deterministic floors; Windows; parallel adapter/batch execution; persisted string-reference index (K6 escalation path); configurable churn window.

## 8. Known unknowns that may create additional issues

| ID | Unknown (DESIGN §19) | Watch point |
|---|---|---|
| K1 | npm package name | resolved with human approval in 35 (ADR-006) |
| K2 | knip across yarn/npm/bun workspace layouts | 08 implementation; may add fixture variants to 33 |
| K3 | vulture output drift | 09 goldens; pin-range narrowing |
| K4 | `go run tool@pin` CI latency | measured in 33; may spawn a caching-docs issue |
| K5 | Windows | out of v1; explicit runner guard (05) |
| K6 | string-verifier performance on huge repos | 15 benchmark criterion; may spawn an index issue |
| K7 | export demotion UX | post-v1 feedback; config knob candidate |
| K8 | comment-preserving TOML editing depth | spiked in 26; may spawn a parser issue |
| K9 (new) | `go.work` multi-module workspaces | flagged in 12; may spawn a discovery extension issue |
| K10 (new) | live-GitHub E2E propose test | optional job specified in 33; may become its own issue |

Discovery of any additional unknown during implementation: file it as a new `docs/issues/NN-*.md` first, then a GitHub Issue (docs remain the source of truth).
