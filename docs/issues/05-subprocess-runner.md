# Title

Hardened subprocess runner (ExecRunner): no shell, env allowlist, timeouts, output caps

## Summary

Implement the single choke point through which reaper spawns every external process (detectors, git, gh, verify built-ins), enforcing the security invariants of DESIGN.md §17 T2/T4.

## Context

Adapters, git, forge, removal, and verify all shell out. Centralizing spawn hardening makes T2 (token leakage) and T4 (resource exhaustion) testable once instead of five times, and gives the conformance kit (07) a seam to assert against.

## Scope

In: `src/exec/runner.ts` + tests.
Out: any specific tool invocation; the `bash -c` verify-command path (27 uses a dedicated, clearly-marked method here).

## Detailed Requirements

1. API:
   ```ts
   type ExecRequest = { cmd: string; args: string[]; cwd: string;
     timeoutMs?: number;            // default 300_000
     maxStdoutBytes?: number;       // default 32 * 1024 * 1024
     env?: Record<string,string>;   // EXTRA vars merged over the allowlist base
     allowToken?: boolean };        // default false; true only for gh (issue 29)
   type ExecResult = { ok: boolean; code: number | null; signal: string | null;
     stdout: string; stderr: string; timedOut: boolean; truncated: boolean; durationMs: number };
   interface ExecRunner { run(req: ExecRequest): Promise<ExecResult>;
     runShell(req: { script: string; cwd: string; timeoutMs: number; env?: Record<string,string> }): Promise<ExecResult>; }
   ```
2. `run`: `child_process.spawn(cmd, args, { shell: false })` — never a shell; `cmd` must not contain path separators unless it is an absolute path that exists (reject `./x; rm` class confusion by construction: args are an array, no interpolation anywhere).
3. Environment construction (T2): child env = allowlist base ∪ `req.env`. Allowlist base (frozen, exact): `PATH, HOME, LANG, LC_ALL, LC_CTYPE, TMPDIR, TERM, GOPATH, GOMODCACHE, GOCACHE, GOFLAGS, GOTOOLCHAIN, PYTHONIOENCODING, VIRTUAL_ENV, UV_CACHE_DIR, NODE_OPTIONS?—NO (exclude NODE_OPTIONS: injection vector), npm_config_cache, CI`. Explicitly NEVER inherited: `GH_TOKEN, GITHUB_TOKEN, REAPER_GITHUB_TOKEN, NPM_TOKEN, NODE_AUTH_TOKEN, NODE_OPTIONS, GIT_ASKPASS` — unless `allowToken: true`, which adds exactly `GH_TOKEN` (from `process.env.GH_TOKEN ?? process.env.GITHUB_TOKEN ?? process.env.REAPER_GITHUB_TOKEN`).
4. Timeout: SIGTERM at `timeoutMs`, SIGKILL 5 s later; result `timedOut: true`, `ok: false`.
5. Output caps: stream-accumulate stdout/stderr; beyond `maxStdoutBytes` (stderr cap fixed 4 MiB) stop buffering, kill as timeout-equivalent, set `truncated: true`, `ok: false`.
6. `ok` = exited normally with code 0 and not truncated. Every completed run logs one debug line `cmd args… (exit, duration)` through the scrubbing logger (02).
7. `runShell` (for issue 27 only): `spawn("bash", ["-c", script])`, same env allowlist WITHOUT token vars unconditionally, cwd required, timeout required. JSDoc marks it "verify-gate only; runs repo-controlled code (ADR-005)".
8. Windows: out of v1 scope (K5) — add a startup guard in this module: `process.platform === "win32"` ⇒ `ReaperError("E_PLATFORM_UNSUPPORTED")`.
9. Tests (behavioral, using real tiny processes — `node -e`): env allowlist (secret var set in parent is absent in child; present with `allowToken`), timeout kill, stdout truncation flag, non-zero exit, `shell: false` metachar safety (arg `"; echo pwned"` arrives literally), scrubbed debug logging.

## Acceptance Criteria

- [ ] All behavioral tests above pass on macOS and Linux CI.
- [ ] Grep test: `child_process` is imported ONLY in `src/exec/runner.ts` (enforced by an eslint no-restricted-imports rule added to the flat config).
- [ ] Token vars set in the parent are unreachable from a child spawned without `allowToken` (test asserts empty).
- [ ] `runShell` never receives token vars even with allowToken-like misuse (type prevents it — no such option).

## Validation

`pnpm test src/exec` green; eslint rule active (`pnpm lint` fails on a synthetic violation, then remove the synthetic file).

## Dependencies

01, 02.

## Non-goals

Retry policies (callers decide), PTY support, Windows (K5).

## Design References

DESIGN.md §17 T2/T4/T5, §9.1 (adapters must use this), §13.3 (runShell consumer); ADR-005.
