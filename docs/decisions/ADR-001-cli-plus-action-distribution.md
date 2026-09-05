# ADR-001: Distribute as a CLI plus GitHub Action, implemented in TypeScript

Status: accepted · Date: 2026-07-10 · Owner: design (Fable)

## Context

reaper must open PRs automatically. Three delivery shapes were considered: (A) CLI + GitHub Action wrapper, (B) CLI only, (C) hosted GitHub App/bot. The choice fixes the authentication model, threat surface, and operational cost. The human user selected (A) on 2026-07-10.

## Decision

1. **CLI first** (`reaper` bin, npm package): every capability works locally and in any CI.
2. **GitHub Action wrapper** (`node20` JS action, committed `dist/` bundle) for zero-boilerplate scheduled adoption. The Action is a thin argv/env mapper around the CLI plus CI-specific security guards (DESIGN.md §15).
3. **No hosted service in v1.** PRs are created with the operator's own token (`GH_TOKEN`), so reaper never holds third-party credentials.
4. **Implementation language: TypeScript on Node ≥20.11, ESM, pnpm, vitest.** Rationale: the heaviest in-process work is TS-ecosystem (knip output, ts-morph removal); GitHub Actions run node natively; the likely contributor audience is JS/TS-first. Go-binary distribution was rejected for v1 because reaper's job is orchestration of external tools, not raw performance, and a Go core would still need Node for ts-morph.
5. Forge operations go through the `gh` CLI rather than an SDK (see ADR-004).

## Consequences

- Zero server operations; the threat model reduces to "code running in the operator's own CI" (ADR-005).
- Users must provision a token with `contents: write, pull-requests: write` for propose mode; report mode needs read-only.
- Node is a hard runtime dependency even for Python/Go-only repos — acceptable: the Action supplies it, and Node is ubiquitous locally.
- A future GitHub App (v2+) can reuse the CLI unchanged; nothing in v1 assumes interactive credentials.
