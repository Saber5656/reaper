# Title

TS/JS entry-point & public-surface heuristics (S6/S8)

## Summary

Implement the TS/JS entry-point pack: detect symbols/files that are invoked by machinery outside the import graph (S6, −0.60) and library public surface (S8, −0.30 when repoKind=library), per DESIGN.md §10.5.

## Context

Framework-invoked code is the classic dead-code false positive: config-referenced handlers, manifest-declared bins, convention-loaded route files. knip covers much of this via its plugins, but reaper double-guards because a PR that deletes a Next.js page is unacceptable (defense in depth per ADR-003).

## Scope

In: `src/signals/entrypoints/types.ts` (pack contract shared with 18/19), `src/signals/entrypoints/ts.ts` + tests.
Out: Python/Go packs (18/19), user `entrypoints` config globs (implemented once in the shared orchestration here, listed in requirements).

## Detailed Requirements

1. Pack contract (`types.ts`): `interface EntrypointPack { language: Language; detect(ctx, findings): Promise<SignalPatch[]> }` + shared helper applying config `entrypoints[]` globs (any finding whose path matches ⇒ S6 with evidence `config:entrypoints`). The shared helper runs once for ALL findings regardless of language (lives here, used by the orchestrator).
2. Built-in TS/JS S6 rules (finding path/symbol matches ⇒ S6 `entrypoint.match`, evidence = rule id + source):
   - `package.json` fields: `main`, `module`, `types`, `browser`, `exports` (all nested string values), `bin` (all values), `scripts` (file-ish tokens `\S+\.(m|c)?js` referenced in script bodies), `files` — resolved relative to the declaring package.json; a finding on a matched file (or a symbol exported from it, for `exports`-mapped entries) fires S6/S8 per rule class.
   - Convention paths (glob, applied to `unused-file` + `unused-export` in them): `pages/**`, `app/**` (next), `src/routes/**` (remix/svelte), `functions/**`, `netlify/functions/**`, `api/**` (vercel), `*.config.{js,ts,mjs,cjs}` at any package root, `.storybook/**`, `**/*.stories.*`, `cypress/**`, `playwright.config.*`, `middleware.{ts,js}`, `instrumentation.{ts,js}`.
   - Symbol conventions: exported names `getServerSideProps, getStaticProps, getStaticPaths, generateMetadata, loader, action, handler, middleware, config` when the file also matches a convention path (both conditions — symbol name alone is too noisy).
   - Declaration-file adjacency: `*.d.ts` findings always S6 (ambient surface).
2b. Rule table lives as data (`const TS_ENTRYPOINT_RULES: Rule[]`) with ids like `ts.pkg.exports`, `ts.conv.next-pages` — ids appear in evidence and fixture expectations.
3. S8 `public-api.surface` (only when `ctx.discovery.repoKind === "library"`): `unused-export` findings whose file is reachable from any `exports`/`main`/`module`/`types` entry (v1 approximation: the file IS one of those entries, or is re-exported by one of them — one hop: parse the entry file's `export ... from "..."` specifiers textually) ⇒ S8. S6 and S8 can co-fire (different rules); scorer just sums.
4. Determinism + caps: package.json parse failures ⇒ warn + skip that rule source; per-rule evidence capped at 3 entries.
5. Tests: rule matrix with a synthetic package (every rule id fires at least once; a non-entry file fires nothing); exports-map nesting (`{"./sub": {"import": "./dist/sub.js"}}`); scripts token extraction; symbol+path conjunction rule (symbol alone must NOT fire); S8 one-hop re-export; repoKind gating (application ⇒ no S8).

## Acceptance Criteria

- [ ] Every rule id in `TS_ENTRYPOINT_RULES` has firing + non-firing tests.
- [ ] Config `entrypoints` glob machinery works for any language's findings (shared helper test with a python path).
- [ ] S8 fires only for libraries; §10.6#3-style score walkthrough reproduced for a TS analog in a scoring integration test (after 21).
- [ ] A Next.js-shaped fixture page with zero imports is NOT proposable end-to-end (final assertion lands in 33's trap suite; rule presence tested here).

## Validation

`pnpm test src/signals/entrypoints/ts*`.

## Dependencies

14 (contract), 06 (repoKind).

## Non-goals

Executing package.json scripts, resolving bundler configs (webpack entries etc. — v2), Python/Go packs.

## Design References

DESIGN.md §10.5 S6/S8, §8 (repoKind), §18.1; ADR-003 (defense in depth).
