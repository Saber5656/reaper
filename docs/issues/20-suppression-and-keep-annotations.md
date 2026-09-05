# Title

Suppression collector: keep-annotations, protect globs, and S9 semantics

## Summary

Implement the user-sovereign suppression layer: `reaper-keep` comment detection on/above declarations, protect-glob matching, and the S9 signal that forces `decision: suppress` regardless of score, per DESIGN.md §10.5/§11.

## Context

ADR-003 makes suppression outrank every score. Users must have a zero-config way (a comment) and a config way (globs) to say "never touch this", and both must be honored before any PR machinery runs. Dismissed-fingerprint suppression (the third channel) arrives with forge dedupe (29) — the policy seam for it is defined here.

## Scope

In: `src/signals/suppression.ts` + tests.
Out: policy evaluation itself (22), PR-dismissal set construction (29).

## Detailed Requirements

1. Keep-annotation detection (S9 `suppression.keep`, no delta, forces suppress via policy):
   - Tokens from `config.keepAnnotations` (default `["reaper-keep"]`), matched case-sensitively inside comments only (lex.ts) — as `<token>` bare or `<token>: <free reason>` (reason captured into evidence).
   - Attachment rules by category:
     - `unused-symbol`/`unused-export`: annotation in the declaration line's trailing comment, or in the 3 lines immediately above `startLine` (blank lines don't break adjacency; another statement does).
     - `unused-file`: annotation anywhere in the file's first 10 lines.
     - `unused-dependency`: annotation as a trailing comment on the manifest line (`go.mod` `require x v1 // reaper-keep`), or for JSON package.json (comments illegal): a sibling key `"//reaper-keep": ["<dep-name>", …]` at the same level as `dependencies` (documented convention); pyproject: trailing `# reaper-keep` on the dependency line.
2. Protect globs: findings whose `location.path` matches any `config.protect` glob get S9 variant `suppression.protect` with evidence = the glob. (Policy 22 treats both S9 variants identically.)
3. Evidence: `path:line token reason?` / `protect:<glob>`.
4. The collector emits `meta` untouched; suppression is expressed ONLY as the S9 signal so `reaper explain` shows exactly why (score still computed for transparency — a suppressed finding shows its would-be score).
5. Tests: attachment matrix per category/language (trailing, above-with-gap, above-broken-by-statement ⇒ no S9; file-head window; go.mod trailing; package.json sibling-key list; pyproject trailing); custom token config; reason capture; protect glob incl. default `**/migrations/**`.

## Acceptance Criteria

- [ ] Full attachment matrix green across ts/python/go declaration styles and all three manifest conventions.
- [ ] A keep-annotated fixture item is suppressed end-to-end with `reasons: ["suppression.keep"]` once 22 lands (cross-ref assertion there and in 33).
- [ ] Suppressed findings still carry a computed score (transparency test after 21).
- [ ] Custom `keepAnnotations` token honored; default token documented in starter config (03's STARTER_CONFIG updated if needed).

## Validation

`pnpm test src/signals/suppression*`.

## Dependencies

14, 15/16 (lex.ts).

## Non-goals

Dismissal-set fetching (29), vulture-whitelist syncing (adapter passthrough only, 09), inline `<token>-next-line` variants (single convention in v1).

## Design References

DESIGN.md §10.5 S9, §11 rule 1, §14.3 (PR suppress instructions must match these conventions); ADR-003 (user sovereignty).
