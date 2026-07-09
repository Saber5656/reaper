# Title

Configuration: reaper.jsonc loader, defaults, and reaper-config.schema.json with safety floors

## Summary

Implement config loading exactly per DESIGN.md §6: JSONC parse, ajv validation with `additionalProperties: false`, shipped defaults, CLI-over-config precedence hooks, and hard safety floors that cannot be configured away.

## Context

Config is a security boundary (DESIGN.md §17 T6): a typo must not silently disable protections, and thresholds must not sink below the safety envelope. Every command consumes `ResolvedConfig`.

## Scope

In: `src/config/types.ts`, `src/config/defaults.ts`, `src/config/load.ts`, `schema/reaper-config.schema.json`.
Out: consuming the config anywhere; `reaper init` generation (04 wires, content spec here).

## Detailed Requirements

1. `src/config/types.ts`: `ReaperConfigInput` (all fields optional — user file) and `ResolvedConfig` (all fields required — post-merge). Field set exactly as the annotated example in DESIGN.md §6 (`version`, `repoKind`, `languages.{ts,python,go}.enabled`, `ignore[]`, `protect[]`, `entrypoints[]`, `keepAnnotations[]`, `actions.{unused-file,unused-dependency,unused-export,unused-symbol}`, `confidence.{proposeThreshold,reportThreshold}`, `budget.{maxFindingsPerPR,maxOpenPRs}`, `verify.{builtin,commands,timeoutSec}`, `pr.{branchPrefix,labels,titlePrefix,draft,retryDismissed}`, `tools.{knip.args,vulture.minConfidence,deptry,goDeadcode,goModDeps}`).
2. `src/config/defaults.ts`: export `DEFAULTS: ResolvedConfig` matching DESIGN.md §6 literal defaults (protect defaults `["**/migrations/**","**/__init__.py"]`, actions defaults propose/propose/report/report, thresholds 0.80/0.20, budgets 10/3, verify builtin true, timeout 900, branchPrefix `reaper/`, labels `["reaper"]`, titlePrefix `chore(reaper): `, draft false, retryDismissed false, vulture minConfidence 60).
3. `src/config/load.ts`: `loadConfig(opts: { cwd: string; configPath?: string }): ResolvedConfig`.
   - Resolution: explicit `--config` path (error if missing) → `<cwd>/reaper.jsonc` → no file = pure defaults.
   - Parse with `jsonc-parser` (`parse` with error collection; any parse error ⇒ `ReaperError("E_CONFIG_PARSE")`, exit 2).
   - Validate with ajv against `schema/reaper-config.schema.json`; any error ⇒ `E_CONFIG_INVALID` listing instancePath + message for every error, exit 2.
   - Deep-merge user input over defaults (arrays replace, never concat; document this in schema descriptions).
4. Safety floors enforced post-merge in code AND in schema where expressible (DESIGN.md §10.2, §17 T6):
   - `confidence.proposeThreshold` ∈ [0.80, 0.99] (schema `minimum: 0.8`).
   - `confidence.reportThreshold` ∈ [0, proposeThreshold) (code check).
   - `budget.maxFindingsPerPR` ∈ [1, 50]; `budget.maxOpenPRs` ∈ [1, 10].
   - `verify.timeoutSec` ∈ [30, 7200].
   - `pr.branchPrefix` must match `/^[A-Za-z0-9._\/-]{1,40}$/` and must not be empty (branch injection guard).
   - `keepAnnotations[]` entries match `/^[A-Za-z0-9_-]{3,40}$/`.
   - Violations ⇒ `E_CONFIG_INVALID`, exit 2 — never clamp silently.
5. Schema: draft 2020-12, `additionalProperties: false` at every object level, every property carries a `description` (source for docs, issue 34). Published at `schema/reaper-config.schema.json`.
6. `configHash(resolved): string` — `sha256` of canonical-JSON (sorted keys) of ResolvedConfig, prefixed `sha256:`; used in scan artifact provenance (§5.1).
7. Starter config content for `reaper init` (consumed by issue 04): a commented JSONC template string exported from `defaults.ts` (`STARTER_CONFIG`), containing `$schema`, all defaults commented out, and a pointer comment to docs.

## Acceptance Criteria

- [ ] No config file ⇒ `loadConfig` returns exact `DEFAULTS` and `configHash` is stable across runs.
- [ ] Unknown key at any nesting level ⇒ exit-2 error naming the path (test: `confidnce`).
- [ ] `proposeThreshold: 0.5` ⇒ rejected; `0.9` accepted; `reportThreshold: 0.95` rejected.
- [ ] JSONC comments and trailing commas parse; malformed JSONC yields `E_CONFIG_PARSE` with line/column.
- [ ] Arrays replace defaults entirely (test `protect: []` clears defaults).
- [ ] Schema file itself compiles under ajv strict mode.

## Validation

`pnpm test src/config` green; run `node dist/cli/main.js` scan-less smoke once 04 lands (deferred wiring noted there).

## Dependencies

01, 02 (errors/log).

## Non-goals

Language auto-detection (06), CLI flag parsing (04).

## Design References

DESIGN.md §6 (schema + defaults), §10.2 (threshold floor), §17 T6; ADR-003 (envelope), ADR-005.
