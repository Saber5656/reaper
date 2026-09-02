# Review resolution record

- Repository: `Saber5656/reaper`
- Pull request: #1
- Parent head observed before this addendum: `c8b666fccbd954f16741d1c887ec05a802493e93`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTNkDOc6Pu_yJ`

### Call ensureLabel before createPr

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_yJ` identifies this contract gap.
- Normative resolution: Make publication order explicit: ensure the configured label exists and is verified before creating a PR that references it; label failure aborts publication without leaving a misleading PR.
- Focused verification before resolving this thread: Run first-publication against a repository without the label and assert label creation/verification precedes PR creation.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkDOc6Pu_yQ`

### Ship knip if the fallback depends on it

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_yQ` identifies this contract gap.
- Normative resolution: Include a pinned, shipped Knip runtime dependency or an equivalent bundled executable and invoke that deterministic local fallback; do not rely on `npx --no-install` finding a package that is not shipped.
- Focused verification before resolving this thread: Run the JS detector in a cold target repository with no local Knip and assert it resolves only the pinned Reaper-provided tool without network installation.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkDOc6Pu_yf`

### Keep uvx probes offline

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_yf` identifies this contract gap.
- Normative resolution: Define doctor probes with an offline flag and a preinstalled/pinned-tool check; a cold cache or missing tool reports unavailable and never downloads from an index.
- Focused verification before resolving this thread: Run doctor with an empty cache and network-deny fixture, then assert no network request or package installation occurs.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkDOc6Pu_yx`

### Keep requirements-based Python deps report-only

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_yx` identifies this contract gap.
- Normative resolution: Classify unused dependencies discovered only from `requirements*.txt` as report-only evidence; they are never eligible for automatic removal or a removable batch.
- Focused verification before resolving this thread: Use a requirements-only Python fixture and assert the finding is displayed but cannot enter a removal proposal.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkDOc6Pu_zE`

### Prevent compileall from dirtying verify worktrees

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_zE` identifies this contract gap.
- Normative resolution: Compile Python batches into an isolated temporary cache/output location outside the verify worktree, then remove it before the clean-tree gate; never write `__pycache__` beside source files.
- Focused verification before resolving this thread: Run verification on a repository without an ignore rule and assert the worktree contains exactly the expected batch files after compile checks.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkDOc6Pu_zQ`

### Don't cap dismissal history at 200 PRs

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_zQ` identifies this contract gap.
- Normative resolution: Paginate all relevant labeled PRs (or use an equivalent complete query) and retain every closed-unmerged dismissal fingerprint; no fixed 200-item limit may affect deduplication.
- Focused verification before resolving this thread: Create more than 200 historical dismissal records with an old matching fingerprint and assert the finding is still suppressed.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkDOc6Pu_zV`

### Filter denied env keys after merging req.env

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_zV` identifies this contract gap.
- Normative resolution: Build the child environment by merging request values and then applying the final denylist, with `GH_TOKEN`, `GIT_ASKPASS`, `NODE_OPTIONS`, and equivalent sensitive keys removed unless the explicit policy allows them.
- Focused verification before resolving this thread: Pass denied keys through both the base environment and `req.env`, then inspect the spawned environment and assert none escape the policy.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkDOc6Pu_zh`

### Reject mutating knip passthrough flags

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_zh` identifies this contract gap.
- Normative resolution: Use an allowlist for non-mutating Knip arguments and reject `--fix`, `--allow-remove-files`, and every documented mutation/removal flag before process launch.
- Focused verification before resolving this thread: Supply mutating and benign flags in configuration and assert mutating flags are rejected while the target checkout remains unchanged.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkDOc6Pu_zw`

### Preserve package-script hits for dependency findings

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_zw` identifies this contract gap.
- Normative resolution: Keep `package.json`'s `scripts` block in the evidence search for dependency findings; any own-file exclusion applies only to non-script metadata that cannot prove usage.
- Focused verification before resolving this thread: Use a fixture where a dependency is referenced only by a package script and assert the finding includes that script evidence.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkDOc6Pu_z7`

### Guard go mod tidy against unbatched removals

- Finding: The existing review thread `PRRT_kwDOTNkDOc6Pu_z7` identifies this contract gap.
- Normative resolution: Run tidy in an isolated copy or snapshot the complete module files, and accept the batch only when the diff removes exactly the requested module set with no collateral requirement changes.
- Focused verification before resolving this thread: Use a go.mod with multiple unused requirements and a one-module batch; assert the extra tidy removals fail the verify gate.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.