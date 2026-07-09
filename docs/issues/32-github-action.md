# Title

GitHub Action: action.yml, bundled node20 entry, security guards, step summary

## Summary

Implement the GitHub Action per DESIGN.md §15: `action.yml` manifest, `src/action/main.ts` mapping inputs→CLI invocations with the fork/`pull_request_target` refusal guards, bot git identity, outputs, and the committed esbuild bundle.

## Context

The Action is the primary adoption surface (user story U1) and carries the sharpest security duty: refusing configurations where untrusted code meets a write token (ADR-005, T1). Guards live in code, not documentation alone.

## Scope

In: `action.yml`, `src/action/main.ts`, esbuild bundling script (`pnpm build:action` → `dist/action.js`, committed), usage-recipe docs stub (full docs 34).
Out: CLI behavior (31), release automation & dist-drift CI check (35), marketplace publishing (35).

## Detailed Requirements

1. `action.yml`: name `reaper`, description (English, mentions confidence-gated dead-code PRs), branding (icon `scissors`, color `purple`); `runs: { using: "node20", main: "dist/action.js" }`; inputs/outputs exactly per DESIGN.md §15 tables (`mode` default `report`, `config`, `fail-on`, `max-prs`, `token` default `${{ github.token }}`, `working-directory` default `.`; outputs `findings-high/medium/low`, `prs-opened`, `report-path`).
2. `src/action/main.ts` (no `@actions/core` dependency — parse `INPUT_*` env + write `GITHUB_OUTPUT` manually; keeps dependency surface per §3.3):
   a. **Guards first** (§15, order matters, each failing with a descriptive error + docs link):
      - `mode=propose` && `GITHUB_EVENT_NAME == "pull_request_target"` ⇒ fail.
      - `mode=propose` && `GITHUB_EVENT_NAME == "pull_request"` && event payload head.repo.full_name ≠ `GITHUB_REPOSITORY` (read `GITHUB_EVENT_PATH` JSON) ⇒ fail (fork).
      - missing token with `mode=propose` ⇒ fail with permissions snippet.
   b. Export `GH_TOKEN` from the `token` input for child gh processes only (05's allowToken path picks it up from process env of the reaper process — set it on `process.env` before pipeline start; ExecRunner still strips it from non-gh children).
   c. Git identity: `git config` in-process invocation params to 28's commit override — `github-actions[bot]` / `41898282+github-actions[bot]@users.noreply.github.com` (§14.1).
   d. Invoke pipelines in-process (import from `src/pipeline`, not subprocess — one bundle): `report` mode = runScan + markdown + step summary + outputs; `propose` mode = runScan (or reuse) + runPropose + outputs.
   e. Outputs written to `GITHUB_OUTPUT`: band counts from artifact stats, `prs-opened` count, `report-path` (artifact copy under `$RUNNER_TEMP/reaper/`).
   f. Exit-code mapping: pipeline exit-3 (`fail-on`) must fail the step; guard failures exit 1 with `::error::` annotation lines.
3. Bundling: `esbuild src/action/main.ts --bundle --platform=node --target=node20 --outfile=dist/action.js` with ts-morph and other lazy imports marked external-and-error-if-reached? No — propose mode needs ts-morph: bundle everything EXCEPT optional lazy strategies get dynamic-import-preserving config; simplest correct v1: full bundle including ts-morph (size acceptable for an action); document bundle size in PR.
4. Usage recipe (README section stub, finalized in 34): schedule + workflow_dispatch on default branch, minimal permissions blocks per mode (§15), checkout + language setup steps example, explicit warning box about propose triggers.
5. Tests: guard matrix via env fixtures (event name/payload combos ⇒ fail/pass) with the pipeline mocked; input parsing (defaults, working-directory chdir); GITHUB_OUTPUT format; identity params passed to commit layer (spy). Bundle smoke: `node dist/action.js` under a report-mode env fixture against a tiny repo executes scan end-to-end (real bundle, fake tools via PATH shim).

## Acceptance Criteria

- [ ] Guard matrix: all three refusal cases fail with actionable messages; same-repo PR report-mode passes.
- [ ] Bundle builds reproducibly (`pnpm build:action` twice ⇒ identical bytes) and the smoke test runs it for real.
- [ ] Outputs appear in `GITHUB_OUTPUT` in correct format; step summary written.
- [ ] No `@actions/*` packages in dependencies (grep test).
- [ ] Live workflow run on this repository (report mode) linked in the PR as evidence.

## Validation

`pnpm test src/action` + bundle smoke + a real `workflow_dispatch` run of the action from the feature branch (`uses: ./`) in this repo.

## Dependencies

30, 31 (pipelines), 28 (identity override), 05 (token env contract).

## Non-goals

Marketplace listing + version tags (35), dist-drift CI enforcement (35), composite-action variant, non-GitHub CI wrappers (docs cover raw CLI).

## Design References

DESIGN.md §15 (spec), §14.1 (identity), §17 T1/T2; ADR-001, ADR-005.
