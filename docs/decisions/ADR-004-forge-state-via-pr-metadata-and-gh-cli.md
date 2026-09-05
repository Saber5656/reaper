# ADR-004: Stateless operation — dedupe/dismissal state lives in PR metadata; forge access via gh CLI

Status: accepted · Date: 2026-07-10 · Owner: design (Fable)

## Context

Idempotent re-runs need memory: which findings are already proposed, and which did a human reject? Options for state: (A) forge PR metadata (labels + fingerprint block in PR bodies); (B) a state file committed to the repo; (C) a server/database. Options for API access: `gh` CLI vs. octokit SDK.

## Decision

1. **State = PR metadata** (DESIGN.md §14.4): every reaper PR carries the `reaper` label and a machine-readable `<!-- reaper:v1 fingerprints=[…] -->` block. Open PR ⇒ don't re-propose; closed-unmerged PR ⇒ fingerprints dismissed (suppress; `pr.retryDismissed` to override); merged ⇒ no record needed. Deterministic branch names make re-runs idempotent even mid-publication.
2. **"Close the PR" is the rejection UX.** No sidecar files to edit, no commands to learn; the reviewer's natural action is the durable signal.
3. **Forge access via `gh` CLI** through the hardened ExecRunner: preinstalled on GitHub runners, ubiquitous locally, keeps token handling (`GH_TOKEN`) inside a battle-tested tool, and keeps reaper's dependency tree smaller (no octokit). The forge module is still an interface (`src/forge/types.ts`) so a GitLab driver can exist later.
4. **Anti-spoofing:** dedupe only trusts PRs matching author `@me` + label `reaper` (DESIGN.md §17 T7); residual risk documented.

## Alternatives rejected

- **(B) committed state file:** pollutes user repos, drifts across branches, creates merge conflicts, and turns every scan into a commit. The baseline-file pattern fits ratcheting linters, not a PR-native tool.
- **(C) server:** contradicts ADR-001.
- **octokit:** more code surface and token plumbing inside reaper for zero v1 capability gain. Revisit only if `gh` absence becomes a real adoption blocker (doctor reports it clearly).

## Consequences

- `gh` becomes a hard runtime dependency for `propose` (not for `scan`/`report`); `reaper doctor` checks it.
- Dedupe reads at most 200 recent reaper PRs — repos exceeding that within a rotation window re-propose old dismissed findings; accepted and documented (budget caps make this slow to matter).
- All state is world-readable in public repos by design; fingerprints contain no secrets (paths + symbol names only — same information as the diff itself).
