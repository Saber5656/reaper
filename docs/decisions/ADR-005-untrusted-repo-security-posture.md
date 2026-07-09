# ADR-005: Security posture — analyzed repos are code-execution surfaces; reaper must never widen that surface silently

Status: accepted · Date: 2026-07-10 · Owner: design (Fable)

## Context

reaper is aimed at public OSS release and runs inside CI adjacent to write-capable tokens. Two facts define the posture: (1) analyzing a repo can execute that repo's code (knip loads `knip.ts`; verify commands run project scripts) — this is irreducible with the chosen detector strategy (ADR-002); (2) reaper's own outputs (PRs) and inputs (tool stdout, PR bodies) cross trust boundaries. Full model: DESIGN.md §17.

## Decision

1. **Name the trust contract instead of pretending isolation:** running reaper on a ref is equivalent to running that ref's dev tooling. Documentation, SECURITY.md, and the Action's guards all encode this. No sandboxing theater in v1.
2. **Hard guards where the contract is commonly violated:** the Action refuses `propose` on fork-PR events and refuses `pull_request_target` outright (DESIGN.md §15). Recommended trigger is `schedule`/`workflow_dispatch` on the default branch.
3. **Least-privilege token flow:** tokens exist only as env (`GH_TOKEN`), reach only `gh` subprocesses, are stripped from all adapter/verify child environments, and are scrubbed from logs. Report mode is documented to run with `contents: read`.
4. **No auto-install, ever:** adapters run exact-pinned tools via `npx --no-install` / `uvx tool==X` / `go run tool@vX`. Missing tools degrade to `skipped`, surfaced by `reaper doctor`.
5. **All parsers treat input as hostile:** subprocess timeouts + output caps, ajv-validated JSON boundaries, size-capped discovery, path-safety module (realpath containment, symlink refusal, `.git`/protect denial) in front of every filesystem mutation.
6. **Safety-relevant config cannot be weakened below floors:** `proposeThreshold ≥ 0.80`, propose-eligibility tied to existing removal strategies, unknown config keys rejected (typos must not silently disable protections).
7. **Own-supply-chain hardening is in-scope for v1** (issue 35): SHA-pinned CI actions, CodeQL, dependabot, dist-matches-src check, npm provenance, SECURITY.md with private disclosure channel.

## Alternatives rejected

- **Sandboxing detectors (containers/seccomp):** platform-specific, heavy, and still defeated by the verify gate's by-design code execution. Honest trust documentation + guards beats partial sandboxes for v1. Revisit for a v2 hosted context.
- **Supporting `pull_request_target` with mitigations:** the pattern is a known token-exfiltration footgun; categorical refusal is simpler than a checklist users will get wrong.

## Consequences

- Some CI recipes users may want (auto-clean PRs *from* fork contributions) are impossible by design in v1.
- Security review of PRs into reaper itself must treat ExecRunner, path-safety, and the Action guards as protected invariants (tests assert them; issue 33/35).
