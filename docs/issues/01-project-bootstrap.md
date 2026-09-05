# Title

Bootstrap TypeScript project: toolchain, lint, test, CI skeleton

## Summary

Create the buildable, lintable, testable empty shell of reaper exactly as specified in DESIGN.md §3.3/§4, so every later issue lands into a working pipeline.

## Context

The repository currently contains only `README.md`. All 34 downstream issues assume this scaffold: pnpm + strict TypeScript ESM, vitest, eslint/prettier, and a CI workflow running lint+typecheck+test.

## Scope

In: package.json, tsconfig, lint/format config, vitest config, `src/cli/main.ts` placeholder printing version, `.gitignore`, LICENSE, CI workflow.
Out: any product logic, action.yml, publishing setup (issue 35).

## Detailed Requirements

1. `package.json`: `"name": "reaper"` (placeholder — publish name resolved in issue 35, K1), `"type": "module"`, `"bin": {"reaper": "dist/cli/main.js"}`, `"engines": {"node": ">=20.11"}`, `"license": "Apache-2.0"`, scripts: `build` (tsc -p tsconfig.build.json), `test` (vitest run), `lint` (eslint . && prettier --check .), `typecheck` (tsc --noEmit), `format` (prettier --write .).
2. Runtime dependencies (exact set, DESIGN.md §3.3): `commander`, `ajv`, `ajv-formats`, `jsonc-parser`, `fast-glob`, `picomatch`. Dev: `typescript` (^5), `vitest`, `eslint` (flat config), `prettier`, `@types/node`, `esbuild`, `@types/picomatch`. `ts-morph` is NOT added here (issue 24 adds it lazily-imported). Pin exact versions in package.json (no `^` for runtime deps), commit `pnpm-lock.yaml`.
3. `tsconfig.json`: `strict: true`, `module: "NodeNext"`, `moduleResolution: "NodeNext"`, `target: "ES2022"`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`, `outDir: "dist"`, `rootDir: "src"`. `tsconfig.build.json` extends it excluding `**/*.test.ts` and `test/`.
4. `src/cli/main.ts`: shebang `#!/usr/bin/env node`, prints `reaper <version from package.json>` and exits 0 (replaced in issue 04).
5. `.gitignore`: `node_modules/`, `dist/` **except** `!dist/action.js` placeholder comment (issue 32 will commit the bundle), `.reaper/`, coverage.
6. `LICENSE`: Apache-2.0 text, copyright line `Copyright 2026 reaper contributors`.
7. `.github/workflows/ci.yml`: triggers `push` (main) + `pull_request`; single job on `ubuntu-latest`; steps: checkout (pinned by SHA), `pnpm/action-setup` (pinned by SHA), setup-node 22 with pnpm cache (pinned by SHA), `pnpm install --frozen-lockfile`, `pnpm lint`, `pnpm typecheck`, `pnpm test`. All third-party actions pinned to full commit SHAs (DESIGN.md §17 T9).
8. `vitest.config.ts`: include `src/**/*.test.ts` and `test/**/*.test.ts`; coverage provider v8 (thresholds set later).
9. eslint flat config: typescript-eslint recommended-type-checked, no-floating-promises as error; prettier as separate check (no eslint-prettier merge).

## Acceptance Criteria

- [ ] `pnpm install --frozen-lockfile && pnpm build && node dist/cli/main.js` prints the version, exit 0.
- [ ] `pnpm lint`, `pnpm typecheck`, `pnpm test` all pass locally (test suite may contain one placeholder test).
- [ ] CI workflow passes on a PR touching this scaffold.
- [ ] All CI actions referenced by full commit SHA; runtime deps exact-pinned; lockfile committed.
- [ ] LICENSE present (Apache-2.0); `.reaper/` gitignored.

## Validation

Run the commands in Acceptance Criteria on a clean clone (`git clean -fdx` first). Verify `git ls-files` contains no `dist/` output except placeholders explicitly listed above.

## Dependencies

None (first issue).

## Non-goals

CLI command surface (04), action bundle (32), publish pipeline & final package name (35).

## Design References

DESIGN.md §3.3 (stack), §4 (layout), §17 T9 (CI pinning); ADR-001.
