# Title

Dependency removal strategy: package.json / pyproject.toml / go.mod manifest edits with lockfile sync

## Summary

Implement the `dependency` removal strategy per DESIGN.md §12.4: surgical manifest edits per ecosystem, lockfile refresh via the detected package manager, never hand-editing lockfiles.

## Context

`unused-dependency` is a default auto-propose category across all three languages — the highest-volume PR type. Manifest edits must preserve formatting/comments and keep lockfiles consistent, or every reaper PR fails the user's CI on lockfile drift.

## Scope

In: `src/removal/dependency.ts` + per-ecosystem editors + tests.
Out: detection (08/10/12), transitive cleanup beyond what the PM does, version bumping.

## Detailed Requirements

1. `canHandle`: `category === "unused-dependency"` (all languages).
2. npm-family (finding path = a package.json):
   - Parse with `jsonc-parser` modification API (format-preserving edits — `applyEdits`); remove the dep key from the table it lives in (`dependencies` or `devDependencies` per knip nativeType).
   - Lockfile sync (PM from discovery, §8): pnpm ⇒ `pnpm install --lockfile-only --ignore-scripts`; npm ⇒ `npm install --package-lock-only --ignore-funding --ignore-scripts`; yarn (berry) ⇒ `yarn install --mode=update-lockfile`; yarn classic / bun ⇒ no reliable lockfile-only mode: per-finding failure `lockfile-sync-unsupported:<pm>` (report-only in practice; documented limitation).
   - `--ignore-scripts` is mandatory on every PM invocation (repo scripts must not execute during removal — §17 T1 narrowing).
   - Also remove the dep from the package.json `"//reaper-keep"` sibling array if present? No — keep-annotated deps never reach removal (suppressed); assert instead.
3. Python (finding path = pyproject.toml):
   - Line-based TOML editor (`src/signals/toml-lite.ts` from 18, extended here with edit support — keep scope: remove one string element from a known array table `[project.dependencies]` / `[project.optional-dependencies.*]` / poetry `[tool.poetry.dependencies]` key-value): matches the dependency by PEP 503-normalized name against the requirement-string head; preserves all other lines byte-identically (K8 fallback approach — if edit-support proves too fragile during implementation, escalate per K8 before adding a dependency).
   - Lockfile refresh: uv ⇒ `uv lock` ; poetry ⇒ `poetry lock --no-update`; pip/no-lock ⇒ nothing. All with scripts-equivalent risks n/a (no hook execution).
4. Go (finding path = go.mod): run `go mod tidy` in the worktree (THE removal, §12.4/§9.7); assert afterward that the target module vanished from go.mod (else per-finding failure `tidy-noop` — signals detection/reality drift, do not ship the edit).
5. Batch semantics: group findings by manifest file; one editor pass + one lockfile sync per manifest; per-finding failure isolation preserved within the group (a failed match doesn't abort siblings).
6. changedFiles includes manifests + lockfiles touched.
7. Tests: format preservation goldens (comments, key order, trailing commas in package.json via jsonc edits; pyproject comments/multiline arrays; poetry table); PEP 503 normalization (`Foo_Bar` requirement vs `foo-bar` finding); each PM sync command via fake exec incl. `--ignore-scripts` assertion; unsupported-PM failure path; go tidy-noop guard; group isolation.

## Acceptance Criteria

- [ ] Byte-exact golden diffs for all three manifest ecosystems (only the target line/key changes).
- [ ] Every PM invocation carries `--ignore-scripts` (or has no script surface) — fake-exec assertions.
- [ ] tidy-noop drift guard fails the finding, not the batch.
- [ ] Real-tool integration on fixtures (33): resulting worktrees pass `pnpm install --frozen-lockfile` / `uv lock --check` / `go build` respectively — tracked in 33, noted here.

## Validation

`pnpm test src/removal/dependency*`.

## Dependencies

23 (chassis), 18 (toml-lite), 06 (PM detection), 08/10/12 (finding shapes).

## Non-goals

requirements.txt editing (deptry findings point at pyproject in v1 scope; plain-requirements repos get report-only — document), peer/optional-peer dependency semantics, monorepo hoisting analysis (knip's workspace attribution is trusted).

## Design References

DESIGN.md §12.4, §9.7, §8 (PM detection), §17 T1/T3, §19 K8.
