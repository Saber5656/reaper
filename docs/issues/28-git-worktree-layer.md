# Title

Git layer: repo state, isolated worktrees, commit authoring, push

## Summary

Implement `src/git/repo.ts` + `src/git/worktree.ts` per DESIGN.md §14.1: read repo state, create/destroy isolated worktrees under `.reaper/worktrees/`, author the batch commit, push the branch — never touching the user's checkout, never force-pushing.

## Context

All mutation happens in worktrees so `propose` is safe on a dirty checkout. Commit/push mechanics are frozen here so the publisher (29) and pipeline (31) compose them without re-deciding git semantics.

## Scope

In: git modules + tests (real git in temp repos — no mocking git itself).
Out: PR creation (29), batching (29), removal (23–26).

## Detailed Requirements

1. `src/git/repo.ts`:
   - `repoInfo(exec, cwd)`: root (`rev-parse --show-toplevel`), headSha, currentBranch, `defaultBranch` resolution: `--base` flag param > config? (no config key in v1 — flag only, §7) > `gh repo view --json defaultBranchRef` when gh available > `git symbolic-ref refs/remotes/origin/HEAD` > error `E_NO_BASE`.
   - `remoteSlug()`: parse `origin` URL (ssh + https forms) → `owner/name`; no origin ⇒ `E_NO_REMOTE` (propose requires a remote; scan does not).
2. `src/git/worktree.ts`:
   - `createWorktree(exec, { root, base, branch })`: `git worktree add --detach .reaper/worktrees/<branch-sanitized> <base>` then `git switch -c <branch>` inside it; existing same-name worktree dir ⇒ remove + recreate (idempotent re-runs, §14.2 determinism); returns `{ dir, cleanup() }`.
   - `cleanup()`: `git worktree remove --force <dir>` + branch deletion if unpushed (`git branch -D`) — called on every failure path (pipeline owns invocation; function must be re-entrant/idempotent).
   - Branch existence handling: if `<branch>` exists locally or on origin (`ls-remote --heads`), reuse semantics = delete local + recreate from base (branch content is derived, deterministic; remote branch is NOT deleted — push updates it fast-forward or fails → E_PUBLISH, 29 handles).
3. Commit authoring (`commitBatch(exec, worktreeDir, batch)`), per §14.1:
   - `git add -A` scoped to the worktree; assert staged set == batch.changedFiles (mirror of 27's gitclean — cheap re-assert).
   - Message: title `chore(reaper): remove <n> dead <category> item(s) [<language>]`; body: one line per finding `<fingerprint> <path>[#<symbol>]`, blank line, `reaper-batch: <batchHash>`.
   - Identity: ambient git identity by default; explicit override params `{ name, email }` (Action sets `github-actions[bot]` / `41898282+github-actions[bot]@users.noreply.github.com`, §14.1 — wired in 32). If no ambient identity and no override ⇒ `E_GIT_IDENTITY` with hint.
   - No commit hooks: `git commit --no-verify` (repo hooks are repo code; propose must not execute them — consistent with §17 T1 narrowing and `--ignore-scripts` in 26).
4. `pushBranch(exec, dir, branch)`: `git -C <dir> push origin <branch>:refs/heads/<branch>` (no force, §14.1); auth via ambient credentials; in Action mode gh has configured auth already; failure ⇒ `E_PUBLISH_PUSH` (exit 5 path).
5. Tests (temp git repos created in test setup): worktree create/reuse/cleanup idempotency; dirty main checkout unaffected by a full create-commit cycle; staged-set assert catches a smuggled file; commit message golden; `--no-verify` asserted via an installed failing pre-commit hook that must NOT block; identity error path; push to a local bare remote; non-ff push failure surfaces `E_PUBLISH_PUSH`.

## Acceptance Criteria

- [ ] Full lifecycle green against real git in CI (Linux+macOS), including hook-bypass and dirty-checkout isolation tests.
- [ ] Re-running createWorktree for the same branch is deterministic and leak-free (`git worktree list` clean after cleanup — asserted).
- [ ] No force flags anywhere (grep test on the module).
- [ ] Push failure maps to exit-5 error class.

## Validation

`pnpm test src/git` (uses real `git`; CI has it).

## Dependencies

05, 02; consumed by 29/31.

## Non-goals

PR creation (29), rebasing existing reaper branches onto new bases (v2 §2.3 auto-rebase), GPG signing (ambient git config may add it; not managed).

## Design References

DESIGN.md §14.1, §14.2 (determinism), §7 (exit 5), §17 T1; ADR-004.
