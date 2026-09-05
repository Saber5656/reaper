# Title

GitHub PR publisher: batching, budgets, PR body with provenance, fingerprint dedupe via gh

## Summary

Implement the forge layer per DESIGN.md §14.2–§14.4: deterministic batch construction, budget enforcement, PR body rendering with the machine-readable fingerprint block, and the dedupe/dismissal reader — all through the `gh` CLI.

## Context

This layer is reaper's public face and its only state store (ADR-004). PR-body fingerprints drive idempotency; closed-PR fingerprints drive permanent dismissal; author+label filtering defends against spoofing (T7).

## Scope

In: `src/forge/types.ts` (Forge interface), `src/forge/github.ts` (gh driver), `src/forge/pr-body.ts`, `src/forge/dedupe.ts`, batching function.
Out: git mechanics (28), pipeline orchestration (31), GitLab driver (v2).

## Detailed Requirements

1. `src/forge/types.ts`: `interface Forge { listReaperPrs(): Promise<ReaperPr[]>; createPr(req): Promise<{ number, url }>; ensureLabel(name): Promise<void> }`; `ReaperPr = { number, state: "open"|"closed"|"merged", fingerprints: string[], headRefName, url }`.
2. Batching (`makeBatches(findings, cfg)`, §14.2): input = decisions `propose`; group `(language, category)`; sort `(path, symbol)`; chunk by `budget.maxFindingsPerPR`; `batchHash = sha256(fingerprints.join("|"))[0:8]`; branch = `${pr.branchPrefix}${category}-${language}-${batchHash}`; deterministic across runs (test: shuffled input ⇒ identical batches).
3. gh driver (`github.ts`) — every call through ExecRunner with `allowToken: true` (05), argv arrays:
   - `listReaperPrs`: `gh pr list --author "@me" --label <cfg label> --state all --limit 200 --json number,state,mergedAt,body,headRefName,url`; merged detection via `mergedAt`; body → fingerprints via the block parser (below).
   - `createPr`: `gh pr create --base <base> --head <branch> --title <t> --label <l...> --body-file <scratch>` (+ `--draft` per config); body via file to avoid argv length limits; scratch outside repo; failure ⇒ `E_PUBLISH_PR`.
   - `ensureLabel`: `gh label create <name> --force`? No — `--force` edits existing labels' colors; use `gh label list --json name` + create-if-absent with fixed color `6e5494` and description "automated dead-code removal by reaper"; race-tolerant (already-exists error swallowed).
4. `pr-body.ts` — renders the §14.3 template exactly: findings table (Path | Symbol | Category | Confidence | Detector); `<details>` per-finding factor breakdown (base + each signal id/delta/first evidence line); Safety section (verified flag + step list from VerifyResult (27) incl. `⚠️ builtin checks only` and `⚠️ unverified` variants; close-to-reject sentence; keep-annotation + protect pointers matching 20's conventions); Provenance line (reaper version, config hash prefix, scan timestamp, `detector@version` list, run URL from `GITHUB_SERVER_URL/+GITHUB_REPOSITORY/actions/runs/GITHUB_RUN_ID` when set else `local`); final line the machine block:
   `<!-- reaper:v1 fingerprints=["a1…","b2…"] -->` (canonical JSON array, sorted).
5. `dedupe.ts`:
   - Block parser: regex `/<!-- reaper:v1 fingerprints=(\[[^\]]*\]) -->/` + JSON.parse + 16-hex validation; unparseable block ⇒ that PR contributes nothing (warn) — hostile bodies must not crash (T7 input posture).
   - `computeDedupe(prs)`: `{ openFingerprints: Set, dismissed: Set }` per §14.4 (open ⇒ skip-at-publish; closed-unmerged ⇒ dismissed; merged ⇒ ignored).
   - Consumed twice: policy ctx `dismissed` (22) and publish-time open-skip.
6. Budget enforcement: `maxOpenPRs − currentOpenReaperPrs` = slots; batches beyond slots are deferred with report annotation `deferred-by-budget` (§14.2); `--max-prs` flag lowers, never raises above config.
7. Tests: batching determinism + chunking; body golden snapshot (full template, fixed inputs) + round-trip (render → parse fingerprints → equal); dedupe matrix (open/closed/merged/hostile-body/foreign-author excluded via `--author @me` argv assertion); label ensure paths; budget slot math incl. pre-existing open PRs; argv/token assertions (only gh calls get allowToken).

## Acceptance Criteria

- [ ] Render→parse round-trip lossless; hostile fingerprint blocks (bad JSON, wrong hex, 10k entries) handled per T7 (cap parse at 500 entries/PR).
- [ ] Dedupe matrix green; dismissed set feeds 22 (integration test with policy).
- [ ] Budget math: config 3, 2 already open, 4 batches ⇒ 1 created + 3 deferred (test).
- [ ] Only gh invocations receive the token (fake-exec sweep assertion).
- [ ] Deterministic batches/branches under input shuffling.

## Validation

`pnpm test src/forge`; live smoke against a scratch GitHub repo deferred to 33's optional live job (documented there).

## Dependencies

02, 03, 05, 22 (dismissed seam), 27 (VerifyResult rendering), 28 (branch/push handoff shape).

## Non-goals

PR updates/rebase of existing open PRs (v2), issue-comment interactions, GitLab (v2), auto-merge (never).

## Design References

DESIGN.md §14.2–§14.4, §17 T7/T8, §7 (exit 5); ADR-004.
