# Title

Verify gate: built-in language checks + user commands in the removal worktree

## Summary

Implement `src/verify/gate.ts` per DESIGN.md §13: after a batch's removals, run built-in syntax/build checks and configured user commands inside the worktree; any failure downgrades the whole batch to report with evidence.

## Context

The verify gate is the last mechanical defense before a PR exists. Its semantics (per-batch, downgrade-not-crash, evidence capture) are part of the safety envelope; its user-command path is a documented arbitrary-code-execution surface (ADR-005) and must stay token-free.

## Scope

In: verify module + tests.
Out: worktree lifecycle (28), batch construction (29/31), per-finding bisect on failure (v2, §13.1).

## Detailed Requirements

1. Signature: `verifyBatch(worktree: string, batch: Batch, cfg, exec, log): Promise<VerifyResult>`;
   `VerifyResult = { ok: true; steps: StepResult[] } | { ok: false; steps: StepResult[]; failedStep: string; evidence: string[] }`;
   `StepResult = { id: string; ok: boolean; durationMs: number; skipped?: string }`.
2. Built-in steps (config `verify.builtin`, default true) — run only those whose precondition holds, in this order (§13.2):
   - `builtin.ts`: `npx --no-install tsc --noEmit` at the batch workspace — precondition: batch language ts/js AND tsconfig.json exists at workspace AND `tsc` resolvable locally; unresolvable ⇒ step recorded `skipped: "no-local-typescript"` (warn).
   - `builtin.python`: `python3 -m compileall -q <dirs of changed files>` — precondition: batch language python AND python3 present.
   - `builtin.go`: `go build ./...` then `go vet ./...` — precondition: batch language go.
   - `builtin.gitclean`: `git status --porcelain` in the worktree parses to EXACTLY the batch's expected changedFiles set (modified/deleted/untracked classification compared; anything unexpected ⇒ fail with the diff as evidence) — always runs, last.
3. User steps (§13.3): for each `verify.commands[i]`: `exec.runShell({ script, cwd: worktree, timeoutMs: verify.timeoutSec * 1000 })` — step id `user.<i>`; runShell is token-free by construction (05).
4. Failure handling (§13.1): first failing step stops the gate; evidence = last 80 lines of combined output (scrubbed logger rules apply); result consumed by the pipeline (31) to downgrade the batch (`reasons += ["verify-failed:<stepId>"]`) and discard the worktree. Timeout/truncation of a step counts as failure with `evidence: ["timeout after …"]`.
5. Language scoping (§13.1): the gate receives one single-language batch (guaranteed by 29's batching); built-in preconditions double-check and skip mismatched languages defensively.
6. All steps' stdout/stderr byte counts and durations land in StepResults (report/PR transparency: 29 renders the verify line from this).
7. Tests: fake-exec matrices — precondition skips, ordered execution, first-failure stop, evidence tail-80 capture + scrubbing (plant a fake token in output), gitclean expected-vs-actual diffs (extra file, missing deletion), user command timeout, empty commands list ⇒ builtin-only, builtin disabled ⇒ user-only, both empty ⇒ ok with zero steps + `steps: []` (pipeline marks the PR "unverified" — flag `verified: steps.some(s=>!s.skipped)` exported).

## Acceptance Criteria

- [ ] Step matrix + failure semantics fully covered; evidence never contains token-shaped strings.
- [ ] gitclean catches both unexpected-extra and unexpected-missing changes (tests).
- [ ] `verified` flag false when every step skipped (consumed by 29's PR body ⚠️ path).
- [ ] Real-fixture run (33): induced test failure downgrades the batch and no PR is attempted — tracked in 33.

## Validation

`pnpm test src/verify`.

## Dependencies

05 (runShell), 03 (config), 23–26 (changedFiles contract).

## Non-goals

Bisecting which finding broke the batch (v2 §2.3), running project test suites by default (opt-in only), sandboxing user commands (trust model ADR-005).

## Design References

DESIGN.md §13 (entire), §17 T1/T2; ADR-005.
