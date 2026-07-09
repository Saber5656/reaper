# Title

reaper doctor: environment, tool, git, token, and config diagnostics

## Summary

Implement `reaper doctor` per DESIGN.md §7: probe every external dependency and precondition, print a status table with actionable hints, honor `--strict-tools`.

## Context

reaper degrades gracefully when tools are missing (adapter `skipped`), which makes a diagnostic command essential — users must be able to see *why* a language produced no findings and what to install.

## Scope

In: `src/cli/commands/doctor.ts` body.
Out: adapter probe implementations (07–12 own them), fixing anything automatically.

## Detailed Requirements

1. Checks, in order, each yielding `{ name, status: "ok"|"warn"|"missing"|"error", detail, hint? }`:
   - `git`: present + repo detected (`rev-parse --git-dir`) + HEAD exists.
   - `config`: reaper.jsonc load result (valid / defaults-only / invalid with first error).
   - `discovery`: language roots found (count per language; warn if a language is enabled but rootless).
   - each adapter's `probe()` from the registry (07): version + chosen runner path in `detail`; missing ⇒ its documented hint (e.g. "install uv: https://docs.astral.sh/uv/").
   - `gh`: `gh --version` + `gh auth status` exit code (warn-only — needed for propose, not scan; token detected via env presence is reported as `env:GH_TOKEN` without printing any part of the value).
   - `node`: version ≥ engines floor.
2. Output: aligned plain-text table to stdout (this command is human-facing; no JSON in v1), one row per check, colored status glyphs (respect `--no-color`).
3. Exit code: 0 if nothing `missing`/`error`; with `--strict-tools`, any adapter-tool `missing` ⇒ exit 4 (`E_TOOL_MISSING_STRICT`); config `error` ⇒ exit 2 regardless.
4. Token safety: the token value must never appear in any output — test greps doctor output after setting a fake `GH_TOKEN` (DESIGN.md §17 T2).
5. Runs without network: all probes are local binary/version checks (adapters' probes already comply).

## Acceptance Criteria

- [ ] Table lists ≥ 9 rows (git, config, discovery, 5 adapters, gh, node) with statuses.
- [ ] Fake-exec matrix test: all-present ⇒ exit 0; vulture missing ⇒ exit 0 with `missing` row; plus `--strict-tools` ⇒ exit 4; broken config ⇒ exit 2.
- [ ] Fake token never appears in output (grep test).
- [ ] Every `missing` row has a non-empty install hint.

## Validation

`pnpm test src/cli/commands/doctor*`; manual run `node dist/cli/main.js doctor` on the dev machine and screenshot into the PR.

## Dependencies

04, 06, 07 (probes exist via 08–12 as they land; doctor renders whatever the registry exposes — implementable right after 07 with partial adapters).

## Non-goals

Auto-installing tools (never, §17 T5), JSON output (v2), checking target-repo test commands.

## Design References

DESIGN.md §7 (contract), §9.1 (probe), §17 T2/T5.
