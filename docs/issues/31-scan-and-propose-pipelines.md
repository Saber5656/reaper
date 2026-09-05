# Title

Pipeline integration: scan and propose orchestration wiring all modules end-to-end

## Summary

Implement `src/pipeline/scan.ts` and `src/pipeline/propose.ts` per DESIGN.md §3.1: the orchestration that composes discovery → adapters → signals → scoring → policy → artifact (scan) and artifact → dedupe → batches → worktree → removal → verify → commit → push → PR (propose), replacing the CLI stubs from 04.

## Context

Every prior issue built a stage; this one makes `reaper scan` and `reaper propose` real. It owns sequencing, failure semantics between stages, `--fail-on`/`--dry-run` behavior, and artifact/dedupe glue — the difference between a parts bin and a product.

## Scope

In: two pipeline modules; scan/propose command bodies; wiring integration tests over synthetic fixtures (real fixture E2E is 33).
Out: any new stage logic (all specified in prior issues), Action wiring (32).

## Detailed Requirements

1. `runScan(ctx): Promise<ScanArtifact>` (§3.1 left column):
   discovery (06) → adapter selection+run (07: statuses + raw) → normalize+corroborate (07) → RepoFileIndex + signal orchestration (14: coverage) → scoring (21) → dedupe fetch IF gh available for dismissed-set (29; unavailable ⇒ empty set + artifact note `dedupe: "unavailable"` — scan works offline) → policy (22) → stats fill → `writeScanArtifact` (30) → summary table stdout.
   - `--only`/`--filter` applied at adapter selection / post-policy respectively (04 flags).
   - `--fail-on <band>`: after artifact write, any non-suppressed finding at/above band ⇒ `E_FAIL_ON` (exit 3, §7).
   - Adapter `failed` statuses: scan continues, exit stays 0 (statuses visible in report) — UNLESS every selected adapter failed ⇒ exit 1 `E_ALL_ADAPTERS_FAILED`.
2. `runPropose(ctx, opts): Promise<ProposeSummary>` (§3.1 right column):
   load artifact (`--scan` or default; missing/stale headSha vs current HEAD ⇒ rerun scan automatically with info log) → forge preflight (gh present, remote slug, base resolution — failures ⇒ exit-5 class BEFORE any mutation) → dedupe (29) → recompute policy with real dismissed set (22 pure ⇒ cheap; keeps scan-then-propose consistent) → batches (29) with open-PR skip + budget slots → per batch sequentially: createWorktree (28) → strategies apply (23–26 via 22's REMOVABLE routing; per-finding failures shrink the batch; empty batch ⇒ skip) → verifyBatch (27; fail ⇒ downgrade batch, cleanup, continue) → commitBatch → `--dry-run` ? (print diffstat + would-be PR body path, cleanup, continue) : pushBranch → createPr (29) → ensureLabel once per run → cleanup worktree on every path (finally).
   - `ProposeSummary`: `{ opened: [{number,url,batch}], deferred, downgraded, dryRun }` printed as a table; also appended into a refreshed artifact (`.reaper/scan.json` decisions updated with `already-proposed:#N` annotations) so `report` reflects reality.
   - Publication errors mid-run: current batch's error recorded, remaining batches still attempted; ≥1 failure ⇒ final exit 5 (partial success visible in summary).
3. Cross-cutting: single ExecRunner instance; totals log line (adapters run, findings, signals fired, duration); `--cwd` respected throughout; all scratch under os tmpdir or `.reaper/`.
4. Integration tests (fake exec for tools/gh/git where needed, real fs):
   - scan happy path over a synthetic multi-language tree with canned adapter outputs → artifact matches golden (fingerprints, scores, decisions).
   - `--fail-on medium` exit-3; all-adapters-failed exit-1; offline (no gh) scan ok with dedupe note.
   - propose dry-run over canned scan: batch/worktree/removal/verify/commit invoked in order (spy sequence), no push/pr calls, worktrees cleaned.
   - verify-failure downgrade path; per-finding removal failure shrinks batch; stale-artifact auto-rescan; partial publish failure exit-5 with one PR opened.
5. Determinism run (§18.3 seed): scan twice on the synthetic tree ⇒ byte-identical artifacts modulo `startedAt` (test normalizes that one field).

## Acceptance Criteria

- [ ] Both pipelines pass the integration matrix above; stage-order spy tests lock the §3.1 sequence.
- [ ] No worktree/scratch leaks after any failure path (`git worktree list` + tmp listing asserts).
- [ ] Exit codes: 0/1/2/3/5 paths each produced by a test through the real CLI entry (`node dist/cli/main.js`).
- [ ] Artifact after propose reflects opened PR numbers (round-trip test).
- [ ] Determinism double-run test green.

## Validation

`pnpm test src/pipeline test/`; then `node dist/cli/main.js scan` dogfooded on the reaper repo itself (expect near-zero findings; paste output in PR).

## Dependencies

All of 02–30 (this is the integration point; minimally runnable once 06,07,14,21,22,30 + at least one adapter land — full DoD needs 23–29).

## Non-goals

Parallel batch processing (sequential v1), resuming interrupted propose runs (worktree recreation makes retry safe), Action specifics (32).

## Design References

DESIGN.md §3.1 (dataflow is the spec), §7, §11, §13.1, §14, §18.3.
