# ADR-002: Orchestrate existing detectors behind an adapter boundary; no own graph engine in v1

Status: accepted · Date: 2026-07-10 · Owner: design (Fable)

## Context

Detection could be built three ways: (A) integrate mature per-language tools (knip, vulture, deptry, x/tools deadcode, `go mod tidy`) behind adapters; (B) build an own cross-language reachability engine (tree-sitter + SCIP/LSIF); (C) hybrid. The human user selected (A) on 2026-07-10. Survey: `docs/research/dead-code-tools-survey.md`.

## Decision

1. v1 detection = five adapters over pinned external tools (DESIGN.md §9). reaper's own analysis investment goes into the **confidence layer** (string-reference verifier, dynamic-usage scanner, entry-point heuristics) — the part no existing tool provides — not into re-deriving reachability.
2. The adapter contract (DESIGN.md §9.1) is the language-extension seam: a new language = a new adapter + entry-point pack + (optionally) removal strategy, with no core changes.
3. Tool versions are exact-pinned; adapters carry golden-output parse tests so upstream format drift breaks reaper's CI, not end users.
4. Native detector semantics are normalized into one Finding schema; native confidence (vulture only) is mapped, never passed through raw (DESIGN.md §10.4).
5. Detector `--fix` capabilities (knip) are never used: removal must pass through reaper's policy + verify gate.

## Alternatives rejected

- **Own engine (B):** highest quality ceiling and consistency, but multi-quarter effort, and it re-implements what knip/deadcode already do well. Wrong risk profile for v1 of a trust-sensitive tool.
- **Hybrid (C):** the corroboration signal S10 already captures the cheap part of hybrid value; a resident graph engine is deferred.

## Consequences

- reaper inherits each tool's blind spots; the confidence layer exists precisely to compensate (DESIGN.md §10.5), and the fixture trap suite (§18.1) proves it.
- Capability is uneven per language (capability matrix §9.2) and must be communicated honestly in reports ("why not auto-fixed" reasons).
- Upstream tools becoming unmaintained is a real risk; the adapter seam bounds the blast radius. Revisit (B) if ≥2 core tools stagnate or precision plateaus.
