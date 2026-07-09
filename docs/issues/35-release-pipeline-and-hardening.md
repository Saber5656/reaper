# Title

Release pipeline and repository hardening: npm publish with provenance, action tagging, dist-drift gate, supply-chain controls

## Summary

Implement the release machinery and repo hardening per DESIGN.md §17 T9: resolve the npm package name (K1), provenance-enabled publish workflow, action version tagging, dist-matches-src CI gate, CodeQL/dependabot/scorecard-class controls.

## Context

reaper ships as both an npm CLI and an action consumed at a git ref — two supply chains. ADR-005 makes hardening a v1 requirement; this issue also closes known unknown K1 (package name).

## Scope

In: release workflow, dist-drift CI job, hardening workflows/configs, package-name resolution + rename sweep, `v0.1.0` release checklist doc.
Out: marketing, docs content (34), any feature code.

## Detailed Requirements

1. **K1 package name:** check npm availability for `reaper` (almost certainly taken); decision ladder: `reaper` → `@<org>/reaper` (if the user creates an org — requires human input; ask before publish) → `reaper-hunter`-class fallback documented for human choice. Update package.json `name` + README install lines; bin stays `reaper` regardless. **This step requires explicit human approval of the final name before any publish** (record the decision in an ADR addendum or `docs/decisions/ADR-006-package-name.md`).
2. Release workflow (`.github/workflows/release.yml`): trigger = tag `v*` push; jobs: full CI (reuse) → `pnpm build && pnpm build:action` → dist-drift assert → `npm publish --provenance --access public` with `id-token: write` (npm Trusted Publishing / OIDC; no long-lived NPM_TOKEN secret if avoidable — document fallback) → create GitHub Release with generated notes → move/create major tag `v0` pointer for action consumers (documented action pinning guidance: consume by full SHA).
3. Dist-drift gate (every PR): rebuild `dist/action.js`, `git diff --exit-code dist/` (T9; 32's committed bundle stays honest).
4. Hardening set (github-oss-repo-hardening alignment; repo rulesets for main already exist per repo owner — verify, don't duplicate):
   - CodeQL workflow (javascript-typescript), weekly + PR.
   - dependabot.yml: npm (weekly, grouped minor/patch), github-actions ecosystems.
   - All workflow `permissions:` blocks explicit + minimal; third-party actions pinned by full SHA (audit existing from 01/33 too).
   - OSSF Scorecard workflow (badge optional).
   - `.github/workflows/` reviewed against `pull_request_target` absence (grep gate in CI — none may appear, ADR-005).
5. Version/pin policy doc (`docs/release-policy.md`): semver rules; adapter tool pins bump = minor; confidence weight/base changes = minor with fixture evidence (ADR-003); schemaVersion bump rules for Finding/config; release checklist (all-issue CI green, e2e green, docs regen clean, dogfood scan on reaper itself clean).
6. Dogfood workflow: scheduled weekly `reaper` action (report mode) running on this repository itself, step summary as the living demo.

## Acceptance Criteria

- [ ] Tag push on a scratch prerelease (`v0.1.0-rc.1`) executes the full pipeline and publishes with provenance to npm (or dry-run `--dry-run` publish if name approval pending — evidence in PR) — human name-approval gate documented and respected.
- [ ] dist-drift gate demonstrably fails on a stale bundle (proof run, reverted).
- [ ] CodeQL, dependabot, scorecard, SHA-pinning, and the `pull_request_target` grep gate all active; workflow permissions audit table in PR.
- [ ] Release-policy doc + checklist committed; ADR-006 (name) merged after human decision.
- [ ] Dogfood workflow runs green on this repo.

## Validation

Scratch-tag pipeline run + gate proof runs, links recorded in the PR; `gh api` audit of repo settings pasted (branch protection, secret scanning, push protection status).

## Dependencies

01 (CI base), 32 (bundle), 33 (e2e in release gate), 34 (docs regen checks in release gate).

## Non-goals

Homebrew/other package managers (v2), signed binaries (npm provenance covers v1), marketplace paid listing.

## Design References

DESIGN.md §17 T5/T9, §19 K1/K4; ADR-005; user's github-oss-repo-hardening conventions (repo already has main protection).
