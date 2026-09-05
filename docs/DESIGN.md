# reaper — Design Document (v1)

Status: authoritative for v1. GitHub Issues are derived from `docs/issues/*.md`; if they disagree with this file, this file wins.
Language note: repository docs are English; the product README may carry a Japanese tagline.

---

## 1. Product overview & positioning

**reaper** is a cross-language dead-code hunter that turns detection into *safe, reviewable action*: it finds project-level dead code in TypeScript/JavaScript, Python, and Go, scores every finding with an explainable confidence value, and automatically opens small, well-evidenced pull requests for the slice of findings that clears a conservative safety envelope. Everything below that bar lands in a triage report instead of a PR.

### 1.1 Differentiation (vs. linters and single-language detectors)

Existing tools stop at detection; none of them owns the last mile:

| Capability | ESLint / tsc | knip | vulture | deadcode (Go) | deptry | **reaper** |
|---|---|---|---|---|---|---|
| Scope | in-file | JS/TS project | Python heuristic | Go whole-program | Python deps | **TS/JS + Python + Go project-level** |
| Unified findings schema |—|—|—|—|—| ✅ one schema, one config |
| Calibrated confidence |—|—| crude % |—|—| ✅ multi-signal, explainable |
| Safety gate before removal |—| ✗ (`--fix` deletes blindly) |—|—|—| ✅ verify gate + policy |
| Opens PRs with provenance |—|—|—|—|—| ✅ budgeted, deduped, revertible |

reaper's three structural bets:

1. **The last mile is the product.** Detection findings rot in CI logs because acting on them is scary manual work. reaper automates removal behind a safety envelope, so dead code actually leaves the repo.
2. **Polyglot integration.** One config, one findings schema, one confidence scale, one PR stream across three ecosystems.
3. **Confidence is the moat.** Multiple independent signals (detector output, repo-wide string references, dynamic-usage patterns, entry-point heuristics, annotations) combine into a score; only the highest-confidence, mechanically-removable categories are auto-proposed. Precision over recall, always: one PR that deletes live code destroys the product's trust.

**Explicit non-territory:** in-file unused variables/imports/parameters. Linters already detect and safely auto-fix those. reaper's floor is cross-module reachability.

### 1.2 Primary user stories

- U1: A maintainer adds the reaper GitHub Action on a schedule; every Monday, up to 3 small PRs arrive deleting unused files and dependencies, each with evidence and a passing verify gate; everything else is in a report artifact.
- U2: A developer runs `reaper scan` locally on a monorepo (TS front-end, Python services, Go tooling) and gets one ranked report of project-level dead code.
- U3: A tech lead gates CI with `reaper scan --fail-on high` to stop new high-confidence dead code from accumulating.
- U4: A reviewer disagrees with a reaper PR, closes it; reaper never re-proposes those findings (close = dismissal).

---

## 2. Goals and non-goals

### 2.1 v1 goals

- G1. CLI (`reaper`) runnable locally and in any CI; GitHub Action wrapper for zero-boilerplate adoption.
- G2. Detection via orchestrated existing tools (knip, vulture, deptry, deadcode, `go mod tidy`) with pinned versions (ADR-002).
- G3. Canonical Finding schema + stable fingerprints across all languages (§5).
- G4. Explainable confidence engine with deterministic scoring (§10) — no ML in v1.
- G5. Policy engine mapping (confidence band × category × config) → `propose | report | suppress` (§11).
- G6. Removal engine: file deletion (TS/JS), TS export/symbol removal, Python symbol removal, dependency removal for all three ecosystems (§12).
- G7. Verify gate: built-in language checks + user-configured commands run in an isolated worktree before any PR (§13).
- G8. PR publication: batched, budgeted, deduplicated PRs with provenance and machine-readable fingerprints; closing a PR dismisses its findings (§14).
- G9. Reports: human Markdown, canonical JSON, GitHub Step Summary (§16).
- G10. Security posture suitable for public OSS release (§17).

### 2.2 v1 non-goals

- In-file lint findings (unused locals/imports/params) — permanently out of scope.
- Own reachability/graph engine (SCIP/LSIF, tree-sitter) — v2 candidate (ADR-002).
- Hosted service / GitHub App — rejected for v1 (ADR-001).
- Forges other than GitHub (GitLab/Bitbucket) — the forge module is an interface, but only the GitHub driver ships in v1.
- Python/Go unused *file* detection; Go symbol *removal* — report-only gaps in v1 (capability matrix §9.2).
- Auto-merge of reaper PRs. Merge is always a human decision.
- LLM-based confidence or removal.
- Windows support (v1 targets Linux + macOS; Windows is a known unknown).

### 2.3 Deferred to v2 (recorded so v1 issues don't accidentally absorb them)

Own import-graph for Python unused files; Go symbol removal via `go/ast` rewriting; staticcheck/depcheck as corroboration detectors; SARIF output; GitLab driver; per-finding bisect on verify failure; coverage-as-negative-signal; auto-rebase of open reaper PRs; fix-batching heuristics learned from merge history.

---

## 3. System architecture

### 3.1 Pipeline data flow

```
                ┌─ adapters (knip │ vulture │ deptry │ deadcode │ go-mod-deps)
discovery ──────┤        │  RawFinding[]
 (repo→targets) └────────▼
                normalize → Finding[] (canonical schema, fingerprint)
                         ▼
                signal collectors (string-ref, dynamic-usage, entrypoints,
                                   suppression, generated, churn)   §10
                         ▼
                confidence scoring → score + band                    §10
                         ▼
                policy engine → decision: propose | report | suppress §11
                         ▼
        ┌────────────────┴───────────────┐
   [report]                        [propose]
   markdown / json /          batch → worktree → removal engine §12
   step-summary §16                → verify gate §13
                                   → commit / push / PR §14
                                   (gate failure ⇒ downgrade to report)
```

`reaper scan` executes discovery → decisions and writes the scan artifact. `reaper propose` consumes the artifact and executes the right-hand branch. All stages are pure functions over explicit inputs except: adapter subprocesses, git operations, forge API calls.

### 3.2 Module map

| Module | Path | Responsibility | Issue |
|---|---|---|---|
| CLI | `src/cli/` | command routing, flags, exit codes | 04 |
| Config | `src/config/` | load/validate `reaper.jsonc`, defaults | 03 |
| Core model | `src/core/` | Finding types, fingerprint, errors, logging | 02 |
| Discovery | `src/discovery/` | language roots, package managers, repo kind | 06 |
| Subprocess | `src/exec/` | hardened spawn wrapper (no shell, timeouts, caps) | 05 |
| Adapters | `src/adapters/` | detector contract, registry, 5 adapters | 07–12 |
| Signals | `src/signals/` | collectors + entrypoint packs | 14–20 |
| Confidence | `src/confidence/` | scoring, bands | 21 |
| Policy | `src/policy/` | decisions | 22 |
| Removal | `src/removal/` | file/symbol/dependency removal strategies | 23–26 |
| Verify | `src/verify/` | built-in checks + user commands in worktree | 27 |
| Git | `src/git/` | repo state, worktrees, commits, push | 28 |
| Forge | `src/forge/` | GitHub driver via `gh`, PR body, dedupe | 29 |
| Report | `src/report/` | markdown / json / step-summary renderers | 30 |
| Pipeline | `src/pipeline/` | scan & propose orchestration | 31 |
| Action | `src/action/` + `action.yml` | GitHub Action entrypoint | 32 |

### 3.3 Runtime & implementation stack (ADR-001)

- Language: TypeScript (strict), ESM only. Node.js `>=20.11`.
- Distribution: npm package exposing bin `reaper`; GitHub Action as `node20` JS action with committed `dist/` bundle (esbuild).
- Package manager for this repo: pnpm (lockfile committed). Test runner: vitest. Lint: eslint + prettier.
- Runtime dependencies (complete list; adding one requires an ADR): `commander`, `ajv` (+`ajv-formats`), `jsonc-parser`, `fast-glob`, `picomatch`, `ts-morph` (TS removal only, lazy-imported).
- External tools invoked (never bundled): `git`, `gh`, `node`/`npx`, `python3`/`uv`, `go`. Presence is probed by `reaper doctor` (issue 13); absence degrades gracefully (§9.1).

---

## 4. Repository layout (canonical)

```
reaper/
├── action.yml                     # GitHub Action manifest (issue 32)
├── dist/action.js                 # committed action bundle (issues 32, 35)
├── package.json  pnpm-lock.yaml  tsconfig.json  eslint.config.js  vitest.config.ts
├── schema/
│   ├── reaper-config.schema.json  # JSON Schema for reaper.jsonc (issue 03)
│   └── finding.schema.json        # JSON Schema for Finding / scan artifact (issue 02)
├── src/
│   ├── cli/main.ts
│   ├── cli/commands/{scan,propose,report,init,doctor,explain}.ts
│   ├── config/{types.ts,defaults.ts,load.ts}
│   ├── core/{finding.ts,fingerprint.ts,errors.ts,log.ts}
│   ├── discovery/workspace.ts
│   ├── exec/runner.ts
│   ├── adapters/{types.ts,registry.ts,knip.ts,vulture.ts,deptry.ts,go-deadcode.ts,go-mod-deps.ts}
│   ├── signals/{types.ts,string-reference.ts,dynamic-usage.ts,suppression.ts,generated.ts,churn.ts}
│   ├── signals/entrypoints/{types.ts,ts.ts,python.ts,go.ts}
│   ├── confidence/{score.ts,bands.ts}
│   ├── policy/decide.ts
│   ├── removal/{types.ts,file-delete.ts,ts-symbols.ts,py-symbols.ts,dependency.ts}
│   ├── removal/py/remove_symbols.py     # LibCST codemod, spawned via uv/python3
│   ├── verify/gate.ts
│   ├── git/{repo.ts,worktree.ts}
│   ├── forge/{types.ts,github.ts,pr-body.ts,dedupe.ts}
│   ├── report/{markdown.ts,json.ts,step-summary.ts}
│   ├── pipeline/{scan.ts,propose.ts}
│   └── action/main.ts
├── fixtures/                      # E2E fixture repos (issue 33)
│   ├── ts-app/   ├── py-app/   └── go-app/
├── test/                          # integration tests; unit tests colocated src/**/*.test.ts
└── docs/                          # this design corpus
```

Unit tests are colocated (`src/**/x.test.ts`); integration/E2E tests live in `test/`.

---

## 5. Canonical data model

### 5.1 Finding (JSON Schema in `schema/finding.schema.json`, issue 02)

```jsonc
{
  "schemaVersion": 1,
  "fingerprint": "9f2c4a1b8e3d5f60",          // §5.2
  "category": "unused-file",                   // unused-file | unused-export | unused-symbol | unused-dependency
  "language": "ts",                            // ts | js | python | go
  "workspace": "packages/web",                 // language-root path relative to repo root; "." for root
  "location": {
    "path": "packages/web/src/legacy/util.ts", // repo-root-relative, POSIX separators
    "startLine": 12,                           // 1-based; null for whole-file / dependency findings
    "endLine": 48,                             // null when unknown
    "symbol": "formatLegacy",                  // null for unused-file; dependency name for unused-dependency
    "symbolKind": "function"                   // function | method | class | variable | type | interface | enum | enum-member | class-member | export | file | dependency
  },
  "detector": {
    "name": "knip",                            // knip | vulture | deptry | go-deadcode | go-mod-deps
    "version": "5.61.0",
    "nativeType": "exports",                   // tool-native issue type / rule code
    "nativeConfidence": null                   // vulture only: 60–100
  },
  "corroboration": [],                         // other detector names reporting the same fingerprint
  "signals": [                                 // appended by collectors, order-independent
    { "id": "string-reference.non-code-hit", "delta": -0.40,
      "evidence": ["templates/mail.html:12"] }
  ],
  "confidence": { "base": 0.70, "score": 0.30, "band": "low" },   // §10
  "decision": { "action": "report", "reasons": ["band=low"] },     // §11
  "meta": { "testOnlyUsage": false, "generated": false }
}
```

The **scan artifact** (`.reaper/scan.json`) wraps findings with run metadata:

```jsonc
{
  "schemaVersion": 1,
  "reaperVersion": "0.1.0",
  "startedAt": "2026-07-10T04:00:00Z",
  "repo": { "root": "/abs/path", "headSha": "abc123", "remote": "github.com/owner/name" },
  "configHash": "sha256:…",                    // hash of resolved config
  "adapters": [ { "name": "knip", "version": "5.61.0", "status": "ok" },
                { "name": "go-deadcode", "status": "skipped", "reason": "no-main-roots" } ],
  "findings": [ /* Finding[] */ ],
  "stats": { "byCategory": {}, "byBand": {}, "byAction": {} }
}
```

`.reaper/` is an ephemeral cache directory; `reaper init` adds it to `.gitignore`.

### 5.2 Fingerprint

`fingerprint = hex(sha256("v1|" + category + "|" + workspace + "|" + location.path + "|" + (symbol ?? "") + "|" + (symbolKind ?? "")))[0:16]`

Properties: stable across line-number shifts and re-runs; changes when a symbol is renamed or moved between files (intended — that is a new claim). Line numbers are display-only. Fingerprints are the dedupe key for PR state (§14.4) and fixtures.

---

## 6. Configuration

File: `reaper.jsonc` at repo root (JSONC; parsed with `jsonc-parser`, validated with ajv against `schema/reaper-config.schema.json`). CLI flags override config; config overrides defaults. Unknown keys are a hard validation error (typo safety).

```jsonc
{
  "$schema": "https://raw.githubusercontent.com/<owner>/reaper/main/schema/reaper-config.schema.json",
  "version": 1,
  "repoKind": "auto",                    // auto | application | library   (§10.6)
  "languages": {
    "ts": { "enabled": true },           // covers js too
    "python": { "enabled": true },
    "go": { "enabled": true }
  },
  "ignore": [],                          // globs: paths never analyzed
  "protect": [                           // globs: analyzed, but never action=propose
    "**/migrations/**", "**/__init__.py"
  ],
  "entrypoints": [],                     // extra globs treated as entry points (kill-switch signal §10.5)
  "keepAnnotations": ["reaper-keep"],    // comment tokens that suppress a finding (§10.5 S9)
  "actions": {                           // per-category policy: "propose" | "report" | "off"
    "unused-file": "propose",
    "unused-dependency": "propose",
    "unused-export": "report",
    "unused-symbol": "report"
  },
  "confidence": { "proposeThreshold": 0.80, "reportThreshold": 0.20 },
  "budget": { "maxFindingsPerPR": 10, "maxOpenPRs": 3 },
  "verify": {
    "builtin": true,                     // tsc/compileall/go build checks (§13.2)
    "commands": [],                      // e.g. ["pnpm test"] — runs via bash -c (§13.3, §17 T2)
    "timeoutSec": 900
  },
  "pr": {
    "branchPrefix": "reaper/",
    "labels": ["reaper"],
    "titlePrefix": "chore(reaper): ",
    "draft": false,
    "retryDismissed": false              // closed-unmerged PR findings stay suppressed
  },
  "tools": {                             // per-adapter overrides
    "knip": { "args": [] },              // extra CLI args appended (validated: no shell metachars needed — arg array)
    "vulture": { "minConfidence": 60 },
    "deptry": {}, "goDeadcode": {}, "goModDeps": {}
  }
}
```

Defaults above ARE the shipped defaults (`src/config/defaults.ts`). A missing `reaper.jsonc` is valid: all defaults apply. `reaper init` writes a commented starter config.

---

## 7. CLI contract

Binary: `reaper`. Global flags: `--config <path>`, `--cwd <path>`, `--log-level error|warn|info|debug` (default `info`), `--no-color`.

| Command | Purpose | Key flags |
|---|---|---|
| `reaper scan` | detect → score → decide; write `.reaper/scan.json`; print summary table | `--json <path>`, `--md <path>`, `--fail-on high\|medium\|low`, `--strict-tools`, `--only <adapter,…>`, `--filter category=…;language=…` |
| `reaper propose` | consume last scan (or run one); apply removals; verify; open PRs | `--max-prs <n>`, `--dry-run` (build branches+diffs locally, no push/PR), `--base <branch>`, `--scan <path>` |
| `reaper report` | re-render reports from last scan artifact | `--json <path>`, `--md <path>` (writes `$GITHUB_STEP_SUMMARY` when set) |
| `reaper init` | detect languages; write `reaper.jsonc` + `.gitignore` entry | `--force` |
| `reaper doctor` | probe external tools, git state, token presence, config validity | `--strict-tools` |
| `reaper explain <fingerprint>` | print full signal/score/decision breakdown for one finding | `--scan <path>` |

**Exit codes (contract, frozen):**

| Code | Meaning |
|---|---|
| 0 | success (findings may exist; default is informational) |
| 1 | unexpected internal error |
| 2 | configuration or usage error (invalid config, bad flags) |
| 3 | `--fail-on` given and ≥1 finding at/above that band |
| 4 | required external tool unavailable while `--strict-tools` |
| 5 | publication failure in `propose` (push/PR API error after removal succeeded) |

Without `--strict-tools`, an unavailable tool marks its adapter `skipped` (visible in report + exit 0). Logs go to stderr; machine output (`--json -` = stdout) is never interleaved with logs.

---

## 8. Workspace discovery (issue 06)

Input: repo root. Output: `DiscoveryResult`:

```ts
type LanguageRoot = { language: "ts"|"python"|"go"; dir: string;   // repo-relative
                      packageManager: "npm"|"pnpm"|"yarn"|"bun"|"pip"|"poetry"|"uv"|"gomod";
                      markers: string[] };                         // files that proved it
type DiscoveryResult = { roots: LanguageRoot[]; repoKind: "application"|"library"; headSha: string };
```

Rules:
- TS/JS root: any directory with `package.json` not under `node_modules`/ignore globs; **only the topmost** is registered — knip handles nested workspaces itself. Package manager from lockfile (`pnpm-lock.yaml` → pnpm, etc.).
- Python root: directory with `pyproject.toml` (or `setup.py`/`requirements*.txt` fallback → packageManager `pip`).
- Go root: directory with `go.mod`.
- Multiple same-language roots are allowed (e.g. `services/a`, `services/b` each with `pyproject.toml`); each becomes a `workspace` value on findings.
- `repoKind` auto-resolution: `library` if (TS root has `package.json` without `"private": true`) OR (`pyproject.toml` has `[project]` with no `tool.poetry.package-mode=false` and a build-system) OR (Go module with no `main` package). Otherwise `application`. Config `repoKind` overrides. Rationale: for libraries, unused *exports* are usually consumed externally → penalty S8 (§10.5).
- Hard caps: discovery scans at most 50,000 directory entries; beyond that, abort with exit 2 and a message to narrow `ignore` globs (DoS guard, §17 T4).

---

## 9. Detection adapters

### 9.1 Adapter contract (issue 07)

```ts
interface DetectorAdapter {
  name: "knip" | "vulture" | "deptry" | "go-deadcode" | "go-mod-deps";
  languages: Language[];
  categories: Category[];
  // Probe availability + version. Never throws; returns status.
  probe(ctx: AdapterContext): Promise<{ available: boolean; version?: string; hint?: string }>;
  // Run detection for one language root. Must not modify the repo.
  run(ctx: AdapterContext, root: LanguageRoot): Promise<RawFinding[]>;
}
type RawFinding = Omit<Finding, "fingerprint"|"signals"|"confidence"|"decision"|"corroboration"|"meta">;
type AdapterContext = { repoRoot: string; config: ResolvedConfig; exec: ExecRunner; log: Logger };
```

Contract rules (enforced by the conformance test kit, issue 07):
- Adapters invoke tools **only** through `ExecRunner` (§17 T3): argv arrays, no shell, cwd = language root, env allowlist, default timeout 300 s, stdout cap 32 MiB.
- Adapters parse tool output defensively: unknown fields ignored, malformed output ⇒ adapter status `failed` with captured stderr head — never a crash, never partial silent results.
- Adapters never read `GITHUB_TOKEN`/`REAPER_GITHUB_TOKEN` (runner strips them from child env).
- Tool version is captured into `detector.version` for provenance; each adapter declares a tested version range and warns outside it.
- An adapter that cannot produce meaningful results in context returns status `skipped` with a machine-readable reason (e.g. `go-deadcode` with no `main` packages).

### 9.2 Capability matrix (frozen for v1)

| Category | TS/JS | Python | Go |
|---|---|---|---|
| unused-file | knip `files` → **removable** | — (v2) | — (v2) |
| unused-export | knip `exports`/`types` → removable (default report, §11) | — | — |
| unused-symbol | knip `classMembers`/`enumMembers` → report-only | vulture → removable (default report) | deadcode → **report-only** |
| unused-dependency | knip `dependencies`/`devDependencies` → removable | deptry DEP002 → removable (prod deps only) | `go mod tidy` diff → removable |

"Removable" = a removal strategy exists (§12); whether it is *proposed* is the policy engine's call (§11).

### 9.3 knip adapter (issue 08)

- Invocation: `npx --no-install knip --reporter json --no-exit-code --include files,exports,types,enumMembers,classMembers,dependencies,devDependencies` at the TS root. knip is a direct dependency of reaper; `--no-install` guarantees no network fetch (§17 T5). If the target repo has its own knip, prefer the repo's version when within the tested range (probe order: repo-local `node_modules/.bin/knip`, then reaper-bundled).
- Parsing: `{ issues: [ { file, <type>: [ { name, line, col } ] } ] }` → RawFindings. Mapping: `files` (the file object itself) → `unused-file`; `exports`/`types` → `unused-export` with symbolKind from type; `enumMembers`/`classMembers` → `unused-symbol` (namespace = enum/class name); `dependencies`/`devDependencies` → `unused-dependency` (path = the package.json declaring it).
- knip respects its own `knip.json`/`knip.ts` config in the target repo (workspaces, ignores) — reaper does not override it; a `tools.knip.args` passthrough exists for edge cases.
- Trust note: loading `knip.ts` executes repo code — inherit trust model §17 T1.

### 9.4 vulture adapter (issue 09)

- Invocation: `uvx vulture==<pinned> <root> --min-confidence <cfg, default 60>`; fallback if `uv` absent: `python3 -m vulture` when importable (probe reports which). Never `pip install` anything (§17 T5).
- Parsing: regex over stdout lines `^(?<path>.+?):(?<line>\d+): unused (?<kind>function|method|class|variable|attribute|property|import) '(?<name>[^']+)' \((?<conf>\d+)% confidence\)$`. `unused import` lines are **discarded** (linter territory, §1.1). Others → `unused-symbol` with `nativeConfidence`.
- Exit code is not trusted for success detection; a parse pass over stdout with ≥0 findings and empty stderr (or known-benign stderr) = success.
- reaper's suppression model is applied downstream (§10.5); vulture whitelists in the target repo are respected by passing them through when present at conventional paths (`whitelist.py`, `.vulture_whitelist.py`).

### 9.5 deptry adapter (issue 10)

- Invocation: `uvx deptry==<pinned> . --json-output <tmpfile>` at the Python root; read tmpfile from the run's scratch dir (never inside the repo).
- Parsing: keep only `error.code == "DEP002"` → `unused-dependency`, `symbol` = module name, `path` = `location.file` (usually `pyproject.toml`).
- Dev-dependency groups are already excluded by deptry for DEP002; therefore Python dep proposals only ever touch production dependency tables (§12.4).

### 9.6 go-deadcode adapter (issue 11)

- Invocation: `go run golang.org/x/tools/cmd/deadcode@<pinned> -json -test ./...` at the Go root (network fetch for the pinned tool module is allowed on first run and documented as such; version is exact-pinned).
- No `main` package in the module ⇒ status `skipped(no-main-roots)`.
- Parsing: `[{ Name, Path, Funcs: [{ Name, Position: { File, Line, Col }, Generated }] }]` → `unused-symbol` (symbolKind `function`/`method` when Name contains a receiver `T.M`), `meta.generated` from `Generated`.
- **testOnlyUsage:** a second run *without* `-test` is executed; findings present only in the no-`-test` run are live-in-tests-only → those are NOT emitted as findings in v1 (conservative), but counted in stats as `testOnlyDead` for the report.
- Go findings are report-only in v1 (no Go symbol removal, §2.2).

### 9.7 go-mod-deps adapter (issue 12)

- Read-only detection (the working tree is never modified; every go invocation carries `GOFLAGS=-mod=readonly`):
  1. `go list -deps -f '{{if not .Standard}}{{.Module.Path}}{{end}}' ./...` computes the used module set.
  2. Direct (non-`// indirect`) `require` entries are parsed from `go.mod`; modules named by `tool` directives are excluded.
  3. Candidates = direct requires ∖ used set; each candidate is cross-checked with `go mod why -m <mod>` — only `(main module does not need module …)` confirms it (capped at 20 candidates per root).
- Each confirmed unused direct requirement → `unused-dependency` finding, base 0.90 (import-graph exact + double-checked), path = `go.mod`.
- Removal strategy for these findings is "run `go mod tidy` for real in the removal worktree" (§12.4).

---

## 10. Confidence engine

### 10.1 Model shape

`score = clamp(base(detector, nativeType, nativeConfidence) + Σ signal.delta, 0.02, 0.99)` — additive, order-independent, fully recorded. Every factor lands in `signals[]` with evidence so `reaper explain` can print the arithmetic. Determinism is a hard requirement: same repo state + config ⇒ identical scores.

### 10.2 Bands

| Band | Score |
|---|---|
| high | ≥ 0.80 |
| medium | 0.50 – 0.79 |
| low | < 0.50 |

`confidence.proposeThreshold` (default 0.80) must equal or exceed the high-band floor; config validation enforces `proposeThreshold ≥ 0.80` — it can be raised, never lowered below high (safety invariant, §17 T6).

### 10.3 Base scores

| Detector · nativeType | Category | Base |
|---|---|---|
| knip `files` | unused-file | 0.80 |
| knip `exports` / `types` | unused-export | 0.70 |
| knip `enumMembers` / `classMembers` | unused-symbol | 0.60 |
| knip `dependencies` / `devDependencies` | unused-dependency | 0.75 |
| vulture (see 10.4) | unused-symbol | 0.45–0.80 |
| deptry DEP002 | unused-dependency | 0.75 |
| go-deadcode | unused-symbol | 0.75 |
| go-mod-deps | unused-dependency | 0.90 |

### 10.4 vulture native-confidence mapping

`base = round(0.45 + (nativeConfidence − 60) / 40 * 0.35, 2)` → 60% → 0.45, 90% → 0.71, 100% → 0.80.

### 10.5 Signals (canonical table — collectors in issues 15–20, scoring in 21)

| ID | Delta | Fires when | Collector |
|---|---|---|---|
| S1 `string-reference.non-code-hit` | −0.40 | identifier appears in a non-code file (html, yml/yaml, json, toml, sql, md, txt, tmpl, jinja, env, ini, csv, xml) | 15 |
| S2 `string-reference.code-string-hit` | −0.30 | identifier appears inside a string literal in any code file other than its own declaration file | 15 |
| S3 `string-reference.comment-only-hit` | −0.05 | identifier appears only in comments elsewhere | 15 |
| S4 `dynamic-usage.same-file` | −0.25 | dynamic-dispatch pattern in the finding's file | 16 |
| S5 `dynamic-usage.same-package` | −0.15 | dynamic pattern elsewhere in the finding's package/module dir (not cumulative with S4; strongest applies) | 16 |
| S6 `entrypoint.match` | −0.60 | finding matches an entry-point heuristic or `entrypoints` config glob | 17/18/19 |
| S7 `generated.file` | −0.20 | file matches generated markers (`@generated`, `Code generated .* DO NOT EDIT`, `*_pb2.py`, `*.gen.ts`, deadcode `Generated:true`) — also sets `meta.generated` | 14 (collector in `signals/generated.ts`) |
| S8 `public-api.surface` | −0.30 | repoKind=library AND (TS: symbol re-exported from an entry in `package.json` `exports`/`main`/`module`/`types`; Python: name in a package `__init__.py` or `__all__`; Go: exported identifier) | 17/18/19 |
| S9 `suppression.keep` | n/a — forces decision `suppress` | keep-annotation comment on/above the declaration, or fingerprint in a dismissed set | 20 |
| S10 `corroboration.agreement` | +0.10 | ≥2 independent detectors emit the same fingerprint | 21 |
| S11 `churn.recent` | −0.10 | finding's file last modified < `14` days ago (git log; configurable window) | 14 (collector in `signals/churn.ts`) |
| S12 `test-only.usage` | 0 (sets `meta.testOnlyUsage`) | non-test references absent but test-file references exist | 15 |

S12 does not change the score; the policy engine downgrades `propose` → `report` on `testOnlyUsage` because removal would also require deleting the referencing tests (out of v1's mechanical-removal scope).

**Signal collectors are additive-only:** they may append signals and set `meta`, never mutate other fields. The scorer (issue 21) is the only writer of `confidence`.

### 10.6 Worked examples (also fixture cases, issue 33)

1. knip reports `unused-file src/legacy/mailer.ts` (base 0.80); no string hits; not entry point → score 0.80 → high → file deletion proposed.
2. vulture 60% on `handle_webhook` (base 0.45); S1 fires (`routes.yml` contains `handle_webhook`) → 0.05 → low → report only. This is the false positive the string verifier exists for.
3. deadcode reports `NewClient` in a Go library-ish module with `main`; repoKind=library → S8 (−0.30) from 0.75 → 0.45 → low. Report only (Go is report-only anyway).
4. knip unused-dependency `lodash` (0.75) + S10 (nothing corroborates; stays 0.75) → medium → **not** proposed even though category=propose, because band < high. Raising precision beats acting on every plausible finding.

---

## 11. Decision policy (issue 22)

Inputs: scored Finding, `ResolvedConfig`, dismissed-fingerprint set (from forge dedupe, §14.4), removal capability matrix (§9.2). Output: `decision.action` + machine-readable `reasons[]`.

Evaluation order (first match wins):

1. `suppress` if S9 keep-annotation, OR path matches `protect` glob, OR fingerprint dismissed (closed-unmerged PR) and `pr.retryDismissed=false`, OR `actions[category] = "off"`.
2. `report` if score < `reportThreshold` → actually **drop** below reportThreshold (default 0.20): findings below it are recorded in stats but excluded from reports (noise floor).
3. `propose` if ALL: band = high AND score ≥ proposeThreshold; `actions[category] = "propose"`; a removal strategy exists for (category, language); `meta.testOnlyUsage = false`; `meta.generated = false`; not in an open reaper PR already (§14.4).
4. else `report`.

Every rejected `propose` precondition is appended to `reasons` (e.g. `["band=high", "no-removal-strategy:go/unused-symbol"]`) — this powers `reaper explain` and the report's "why not auto-fixed" column.

---

## 12. Removal engine

### 12.1 Contract (issue 23)

```ts
interface RemovalStrategy {
  id: "file-delete" | "ts-symbol" | "py-symbol" | "dependency";
  canHandle(f: Finding): boolean;
  // Mutates ONLY inside worktreeRoot. Returns changed files or a typed failure.
  apply(worktreeRoot: string, findings: Finding[]): Promise<RemovalResult>;
}
type RemovalResult = { ok: true; changedFiles: string[] } | { ok: false; failed: { fingerprint: string; reason: string }[] };
```

Path-safety invariants (enforced centrally, tested with hostile fixtures — §17 T3):
- Every target path is resolved with `realpath` and must remain inside the worktree root.
- `lstat` first: if the path itself is a symlink, refuse (delete of a symlink *file* finding is allowed only as unlink of the link, never following it; v1 simply refuses symlinked findings).
- `.git/**`, `protect` globs, and anything outside the finding's workspace are refused.
- A strategy failure on one finding removes that finding from the batch (and its PR) — it never aborts sibling findings.

### 12.2 File deletion (issue 23) — TS/JS `unused-file`

`git rm` equivalent (unlink + stage). Refuses: files with side-effectful-looking top-level await/register patterns? — No: that judgment already happened in scoring (entry-point + string signals). The strategy is mechanical.

### 12.3 Symbol removal

- **TS (issue 24, ts-morph):** remove the exported declaration for `unused-export` when the symbol has no internal references; if it does have internal references, remove only the `export` modifier (demote to local) — knip semantics ("export is unused") make demotion the correct minimal edit; if the demoted local then becomes unused, that's a future finding. Re-export chains (`export { x } from "./y"`) remove the re-export specifier only. After edits: organize nothing else; verify file still parses (ts-morph diagnostics).
- **Python (issue 25, LibCST):** `src/removal/py/remove_symbols.py` receives a JSON plan `{file, symbol, kind, line}` list on stdin, removes matching `FunctionDef`/`ClassDef`/assignment targets, writes files in place (inside worktree), prints JSON result. Invoked as `uv run --with libcst==<pinned> python3 src/removal/py/remove_symbols.py` (fallback plain `python3` if libcst importable). Line number must match the def's line ±2 (guards against same-name shadowing). Also removes now-empty trailing `__all__` entries naming the symbol.

### 12.4 Dependency removal (issue 26)

- npm-family: edit `package.json` (remove the key from the right table), then run the detected package manager's install to sync the lockfile (`pnpm install --lockfile-only` / `npm install --package-lock-only` / `yarn` mode table in issue 26). Never hand-edit lockfiles.
- Python: remove from `[project.dependencies]` / poetry table (TOML surgical edit preserving comments — `tomlkit`? No new JS dep: implement minimal line-based TOML array editor specified in issue 26; uv/poetry lock refresh commands per manager, run only if lockfile exists).
- Go: run `go mod tidy` in the worktree (this is the removal).

---

## 13. Verify gate (issue 27)

Runs inside the candidate worktree after removal, before commit finalization.

### 13.1 Semantics

- Gate failure ⇒ the whole batch is downgraded to `report` with `reasons += ["verify-failed:<step>"]` and evidence = last 80 lines of output; the worktree is discarded; **no PR**. (Per-finding bisect = v2.)
- Gate is per-batch, and batches are per-language (§14.2), so a Python test failure never blocks a TS batch.

### 13.2 Built-in checks (config `verify.builtin`, default true; each runs only if its precondition holds)

| Language | Check | Precondition |
|---|---|---|
| ts/js | `tsc --noEmit` (repo's own typescript via `npx --no-install tsc`) | tsconfig.json exists && typescript resolvable |
| python | `python3 -m compileall -q <changed files' dirs>` | python3 present |
| go | `go build ./...` then `go vet ./...` | always (go root) |
| all | `git status --porcelain` sanity: only expected files changed | always |

### 13.3 User commands

`verify.commands[]` run sequentially via `bash -c <cmd>` with cwd = worktree root, timeout `verify.timeoutSec`. These execute repo-configured code by definition — same trust level as running the project's tests (§17 T1/T2). Environment: inherited allowlist minus tokens.

---

## 14. Git & PR publication

### 14.1 Git layer (issue 28)

- Preconditions for `propose`: clean working tree not required (reaper never touches the user's checkout) — all mutation happens in `git worktree add` under `.reaper/worktrees/<branch>`; base = `--base` flag → config → remote default branch.
- Commit format: one commit per PR. Message: `chore(reaper): remove <n> dead <category> item(s) [<language>]` + body listing `fingerprint path symbol` lines + `Co-Authored-By: reaper <reaper@invalid>`? — No: commit author/committer = ambient git identity; in Action mode `src/action/main.ts` sets `git -c user.name="github-actions[bot]" -c user.email="41898282+github-actions[bot]@users.noreply.github.com"`. No Co-Authored-By trailers.
- Push: `git push origin <branch>` via ExecRunner with `GH_TOKEN`-based credential only in Action mode (gh sets up auth); locally, user's existing credentials. Never `--force`.

### 14.2 Batching (issue 29)

- Input: findings with `action=propose`. Group by `(language, category)`; sort deterministically by `(path, symbol)`; chunk into `budget.maxFindingsPerPR` (default 10).
- Open at most `budget.maxOpenPRs` (default 3) minus currently-open reaper PRs; leftover batches are reported as "deferred by budget".
- Branch name: `{pr.branchPrefix}{category}-{language}-{sha256(joined fingerprints)[0:8]}` → `reaper/unused-file-ts-a1b2c3d4`. Deterministic: same batch ⇒ same branch (idempotent re-runs).

### 14.3 PR content (issue 29)

Title: `{pr.titlePrefix}remove {n} unused {category-noun} ({language})`. Body template (canonical):

```markdown
## 🪦 reaper: {n} dead-code removal(s) — confidence ≥ {threshold}

| Path | Symbol | Category | Confidence | Detector |
|---|---|---|---|---|
…one row per finding…

<details><summary>Evidence & scoring breakdown</summary>
…per finding: base, each signal with delta + evidence…
</details>

### Safety
- Verify gate: {✅ commands run | ⚠️ builtin checks only} — {list}
- To reject: **close this PR** — reaper will never re-propose these findings.
- To protect code paths permanently: add a `reaper-keep` comment or a `protect` glob in `reaper.jsonc`.

### Provenance
reaper {version} · config {sha256[0:12]} · scan {timestamp} · detectors: {name@version…} · run: {CI url|local}

<!-- reaper:v1 fingerprints={json array} -->
```

The HTML comment is the machine-readable dedupe record; parser tolerates surrounding edits.

### 14.4 State & dedupe via forge metadata (ADR-004, issue 29)

No server, no committed state file. Before opening PRs:

1. `gh pr list --label reaper --state all --limit 200 --json number,state,mergedAt,body,headRefName`.
2. Fingerprints in **open** reaper PRs → decision stays `propose` but publication skips them (`reasons += ["already-proposed:#123"]`).
3. Fingerprints in **closed & unmerged** reaper PRs → dismissed set → `suppress` (unless `pr.retryDismissed`).
4. Merged PRs need no handling (the code is gone; a reappearing fingerprint is a genuinely new claim).

GitHub interaction is exclusively through `gh` (preinstalled on GitHub runners, ubiquitous locally): `gh pr list/create/comment`, `gh api`. Token: `GH_TOKEN` env read by gh itself; reaper never persists or logs it (§17 T2).

---

## 15. GitHub Action (issue 32)

`action.yml` at repo root; `runs: { using: "node20", main: "dist/action.js" }` (committed esbuild bundle; CI job asserts dist == build(src), issue 35).

| Input | Default | Meaning |
|---|---|---|
| `mode` | `report` | `report` (scan + step summary + artifact) or `propose` |
| `config` | `reaper.jsonc` | config path |
| `fail-on` | `""` | forwards to `--fail-on` |
| `max-prs` | config | budget override |
| `token` | `${{ github.token }}` | exported as `GH_TOKEN` for gh |
| `working-directory` | `.` | monorepo subdir support |

Outputs: `findings-high`, `findings-medium`, `findings-low`, `prs-opened`, `report-path`.

Hard security guards baked into `src/action/main.ts` (§17 T1):
- If `mode=propose` AND event is `pull_request`/`pull_request_target` AND head repo ≠ base repo ⇒ fail with an explanatory error (never analyze fork code with a write token).
- If `mode=propose` and event is `pull_request_target` at all ⇒ fail (unsafe pattern, full stop).
- README documents the minimal permissions block: `permissions: { contents: write, pull-requests: write }` for propose; `contents: read` for report.
- Recommended trigger in docs: `schedule` + `workflow_dispatch` on the default branch.

Checkout, language toolchains, and dependency install are the *workflow author's* steps (documented recipe); the action never installs project dependencies itself.

---

## 16. Reporting (issue 30)

- **JSON:** the scan artifact itself (§5.1) — canonical machine output, schema-validated.
- **Markdown:** ranked table grouped by action (proposed / reported / suppressed-counts), each row `path · symbol · category · language · score · band · top negative signal`, plus adapter status table and "why not auto-fixed" reasons. Deterministic ordering.
- **GitHub Step Summary:** the Markdown report written to `$GITHUB_STEP_SUMMARY` when present.
- `reaper explain <fp>`: full arithmetic for one finding (base, every signal + evidence lines, decision reasons).

---

## 17. Security model

Assume public OSS release; assume reaper runs inside other people's CI with a write-capable token nearby. Full user-facing policy in issue 34 (SECURITY.md); this section is the engineering source of truth.

### 17.1 Trust boundaries

| Boundary | Trust level |
|---|---|
| Analyzed repo content (code, configs, knip.ts, verify commands) | **Untrusted by reaper, trusted by the operator.** Running reaper on a ref ≡ executing that ref's dev tooling (knip config load, verify commands). reaper must make this explicit and never widen it silently. |
| Detector tool binaries | Pinned versions; invoked with argv arrays; treated as semi-trusted (their output is untrusted input to parsers). |
| Forge (GitHub) responses | Untrusted input to parsers (PR bodies parsed for fingerprints are attacker-writable in public repos — see T7). |
| Tokens | Secret; env-only; never in argv, logs, files, or child env except `gh`. |

### 17.2 Threats & controls

| ID | Threat | Control |
|---|---|---|
| T1 | Fork-PR code execution with write token (poisoned `knip.ts`, `package.json` scripts, verify commands) | Action guards (§15): propose mode refuses fork-PR and `pull_request_target` events; docs mandate schedule/push triggers; report mode documented to run with read-only token. |
| T2 | Token exfiltration via detector/verify subprocesses | ExecRunner env allowlist (`PATH,HOME,LANG,LC_*,TMPDIR,GOPATH,GOMODCACHE,GOCACHE,PYTHONPATH?—no: minimal fixed list in issue 05`); `GH_TOKEN` injected only into `gh` invocations; log scrubber redacts token-shaped strings. |
| T3 | Path traversal / symlink escape during removal (malicious finding paths from a compromised tool or crafted repo) | Central path-safety module (§12.1): realpath containment, lstat symlink refusal, `.git` and protect-glob denial; hostile-fixture unit tests. |
| T4 | Resource exhaustion (parser bombs, giant repos, tool hangs) | Subprocess timeouts (300 s default) + stdout caps (32 MiB) + discovery entry cap (§8) + report row caps with explicit "truncated" markers. |
| T5 | Supply-chain: adapters auto-installing tools at run time | Never auto-install: `npx --no-install`, `uvx tool==pinned`, `go run tool@pinned` (exact versions); reaper's own deps minimal + lockfile + npm provenance on publish (issue 35). |
| T6 | Unsafe configuration weakening the envelope | Schema hard limits: `proposeThreshold ≥ 0.80`; `actions.*` cannot enable propose for (category,language) pairs without removal strategies; unknown config keys rejected. |
| T7 | Spoofed dedupe records (attacker opens a fake "reaper PR" whose fingerprint block suppresses real findings) | Dedupe only trusts PRs whose author is the authenticated user/bot identity (`gh pr list --author @me`) AND carry the reaper label; documented residual risk in repos where others can use the same bot identity. |
| T8 | PR spam / runaway automation | Budgets (`maxOpenPRs`, `maxFindingsPerPR`), deterministic branch names (idempotent re-runs), propose never runs implicitly (explicit command/mode). |
| T9 | reaper's own repo compromise → malicious action | Repo hardening (issue 35): branch protection (already on), CodeQL, dependabot, pinned CI actions by SHA, dist-matches-src CI check, npm provenance, SECURITY.md disclosure policy. |

### 17.3 Secure defaults summary

report-only Action mode; propose limited to high-band unused-file/unused-dependency; budgets on; verify builtin on; no auto-install; no telemetry of any kind; logs to stderr with token scrubbing.

---

## 18. Quality strategy

### 18.1 The precision hard gate (issue 33)

Three fixture repos (`fixtures/ts-app`, `fixtures/py-app`, `fixtures/go-app`) seeded with:
- genuinely dead items (expected: found, correct category, band floor asserted),
- **alive-but-tricky traps**: string-keyed registry handlers, `getattr` targets, framework entry points, template-referenced symbols, reflection users, `__all__` exports, generated files (expected: `mustNotPropose` — any trap with `decision.action=propose` fails CI hard),
- keep-annotated items (expected: suppressed).

`fixtures/*/expected.json` maps fingerprint inputs → expectations; the E2E harness runs the real pipeline (real tools, installed in CI) and diffs. This gate is the product's precision contract: **zero traps proposed, ever.**

### 18.2 Test taxonomy

| Layer | Scope | Where |
|---|---|---|
| Unit | pure modules (fingerprint, scoring, policy, parsers with golden tool outputs, path safety with hostile inputs) | `src/**/*.test.ts` |
| Adapter golden | each adapter parses a committed real tool-output sample; version drift breaks here first | `src/adapters/*.test.ts` + `test/golden/` |
| Integration | pipeline over fixtures without forge (propose `--dry-run`) | `test/` |
| E2E | full scan+propose dry-run in CI with real tools on fixtures + precision gate | `test/e2e/` (issue 33) |
| Publication | PR body render + dedupe parser round-trip (no live API; `gh` mocked at ExecRunner seam) | `src/forge/*.test.ts` |

### 18.3 Determinism check

CI runs `reaper scan` twice on fixtures and asserts byte-identical scan artifacts (minus timestamps) — guards the "same input ⇒ same score" invariant (§10.1).

---

## 19. Known unknowns (tracked; may spawn issues during implementation)

| # | Unknown | Trigger to resolve |
|---|---|---|
| K1 | npm package name availability ("reaper" is likely taken) | resolved in issue 35 before publish; bin stays `reaper` |
| K2 | knip behavior matrix across monorepo layouts (yarn/npm/bun workspaces) beyond pnpm | during issue 08; may add fixture variants |
| K3 | vulture output-format drift across versions | golden tests in issue 09 will catch; pin range may narrow |
| K4 | `go run tool@version` cold-start latency in CI (module download) | measure in issue 33; may pre-warm via setup-go cache docs |
| K5 | Windows support (paths, bash -c verify commands) | explicitly out of v1; revisit on demand |
| K6 | Very large repos: string-reference verifier performance (repo-wide grep per finding) | issue 15 includes a batching design + benchmark acceptance criterion; may need an index |
| K7 | Whether demoting `export` (vs deleting declaration) surprises users | usability feedback post-v1; config knob candidate |
| K8 | tomlkit-equivalent comment-preserving TOML editing in TS without new deps | spiked in issue 26; fallback = line-based editor with strict scope |

---

## 20. Glossary

| Term | Meaning |
|---|---|
| finding | one canonical dead-code claim (schema §5.1) |
| fingerprint | stable 16-hex identity of a finding (§5.2) |
| signal | one scored evidence item appended by a collector (§10.5) |
| band | high / medium / low score bucket (§10.2) |
| decision / action | propose (open PR) / report / suppress (§11) |
| batch | findings sharing one PR (§14.2) |
| verify gate | post-removal checks that must pass before publication (§13) |
| trap | fixture item that is alive but looks dead; must never be proposed (§18.1) |
| dismissed | fingerprint from a closed-unmerged reaper PR; never re-proposed (§14.4) |
