# Title

Workspace discovery: language roots, package managers, repoKind resolution

## Summary

Implement `src/discovery/workspace.ts` producing `DiscoveryResult` (language roots with package managers, repoKind, headSha) per DESIGN.md §8, with the directory-entry cap DoS guard.

## Context

Every adapter runs per language root; scoring signal S8 depends on `repoKind`; findings carry `workspace`. Discovery is the pipeline's first stage and must be deterministic and bounded.

## Scope

In: discovery module + tests with synthetic fixture trees.
Out: JS workspace *internal* traversal (knip's job, §9.3), git worktree logic (28).

## Detailed Requirements

1. Types exactly per DESIGN.md §8 (`LanguageRoot`, `DiscoveryResult`).
2. Walk: iterative BFS from repo root using `fast-glob`-free manual readdir (control + cap); skip: `node_modules`, `.git`, `dist`, `build`, `.venv`, `venv`, `__pycache__`, `vendor`, `.reaper`, plus config `ignore` globs (picomatch). Count every visited entry; > 50,000 ⇒ `ReaperError("E_DISCOVERY_TOO_LARGE")` exit 2 with hint to add `ignore` globs.
3. Root detection rules (§8):
   - TS/JS: dir contains `package.json`; register only the **topmost** such dir per subtree (skip descendants — knip traverses workspaces itself). `language: "ts"` (single value covers js; Finding.language refinement happens in adapters). PM detection: `pnpm-lock.yaml`→pnpm, `yarn.lock`→yarn, `bun.lockb`/`bun.lock`→bun, `package-lock.json`→npm, none→npm.
   - Python: dir contains `pyproject.toml` (PM: `uv.lock`→uv, `poetry.lock`→poetry, else pip) OR `setup.py`/`requirements.txt`(+variants) → pip. Register topmost per subtree.
   - Go: dir contains `go.mod` → gomod. Register every `go.mod` dir (Go multi-module repos are flat, not nested workspaces; nested `go.mod` under another Go root is registered too).
   - `markers`: the exact filenames that proved detection.
   - Language disabled in config ⇒ roots of that language are not emitted.
4. `repoKind` auto-resolution (§8, evaluated only when config `repoKind: "auto"`):
   - `library` if ANY: (a) a TS root's package.json lacks `"private": true` AND has `name` + (`main`|`module`|`exports`|`types`); (b) a Python root's pyproject has `[build-system]` AND `[project].name`; (c) a Go root has no `package main` file (probe: grep `^package main` across `*.go` capped at first hit).
   - else `application`.
5. `headSha`: `git rev-parse HEAD` via ExecRunner; absence of git ⇒ `E_NOT_A_REPO` exit 2 (reaper requires a git repo).
6. Determinism: roots sorted by `(language, dir)`.
7. Tests: synthetic trees in temp dirs covering — monorepo with nested package.json workspaces (only topmost registered); dual Python services; Go multi-module; disabled language; cap breach (generate 50k+ entries cheaply with many empty dirs — keep runtime < 10 s); each repoKind branch; ignore-glob exclusion.

## Acceptance Criteria

- [ ] All rule-matrix tests pass; ordering deterministic.
- [ ] Cap breach exits 2 with the hint message.
- [ ] `repoKind` matrix: private app package → application; publishable package.json → library; pyproject with build-system → library; go module with main → application (given no other triggers).
- [ ] Runs on the reaper repo itself and returns its own TS root (dogfood smoke test).

## Validation

`pnpm test src/discovery`; run compiled discovery against `fixtures/` trees once issue 33 lands (cross-check noted there).

## Dependencies

01, 02, 03, 05.

## Non-goals

Detecting frameworks (17–19), symlinked roots (refused implicitly by walk not following symlinks — document in code).

## Design References

DESIGN.md §8; §10.5 S8 (repoKind consumer); §17 T4 (cap).
