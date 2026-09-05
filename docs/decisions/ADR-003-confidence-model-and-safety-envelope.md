# ADR-003: Deterministic additive confidence model; auto-PR only inside a conservative safety envelope

Status: accepted · Date: 2026-07-10 · Owner: design (Fable)

## Context

reaper's tagline promises "確信度付きで自動PR" (auto-PRs with confidence). One PR that deletes live code destroys user trust permanently, so the scoring/action design is the product's core risk decision. Options: (A) deterministic explainable scoring (base per detector + signed signal deltas); (B) ML/LLM-scored confidence; (C) pass through native tool confidences.

## Decision

1. **Deterministic additive model** (DESIGN.md §10): `score = clamp(base + Σ deltas)`, every factor recorded with evidence, `reaper explain` reproduces the arithmetic. Same repo state + config ⇒ identical output (CI-enforced, §18.3).
2. **Precision over recall, structurally:**
   - Auto-propose requires band `high` (≥0.80) AND category enabled AND a removal strategy AND no `testOnlyUsage`/`generated` flags (§11).
   - Default propose categories: `unused-file` (TS/JS), `unused-dependency` (all languages). Exports/symbols default to report and are opt-in.
   - `proposeThreshold` can be raised but never configured below 0.80 (schema-enforced).
   - Findings referenced only by tests are never proposed (removal would orphan tests).
3. **Independent second opinion:** the string-reference verifier (S1–S3) greps the whole repo — including non-code files — for each candidate identifier. It is detector-independent, language-agnostic, and exists to catch exactly the dynamic-usage class of false positives that kills naive tools.
4. **Suppression is user-sovereign:** keep-annotations, protect globs, and "close the PR = dismissed forever" (ADR-004) all outrank any score.
5. No ML/LLM in the v1 scoring path.

## Alternatives rejected

- **(B) ML/LLM scoring:** non-deterministic, unexplainable to reviewers, needs labeled data we don't have, and makes the precision gate untestable. Possible v2 *re-ranking* layer on top of deterministic floors.
- **(C) Native confidences:** only vulture has one and it is per-construct heuristic; cross-tool comparability would be fictional.

## Consequences

- Scores are calibrated by design review + fixture traps, not statistics; the weights table (§10.5) is expected to be tuned via fixture evidence over time — changing a weight requires a fixture case demonstrating why.
- Recall is deliberately sacrificed: many true dead findings sit in `report` forever unless the user opts categories up. That is the intended posture.
- The fixture precision hard gate (§18.1) is the enforceable form of this ADR: zero alive traps proposed, ever.
