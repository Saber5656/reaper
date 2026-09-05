# Title

CLI skeleton: command routing, global flags, exit-code contract, init command

## Summary

Implement the `reaper` CLI surface with commander: six subcommands wired to stub pipeline entry points, global flags, the frozen exit-code contract, and the fully functional `reaper init`.

## Context

DESIGN.md §7 freezes the command/flag/exit-code contract. Later issues fill the command bodies (13 doctor, 30 report, 31 scan/propose); this issue makes the surface real so integration lands incrementally.

## Scope

In: `src/cli/main.ts`, `src/cli/commands/{scan,propose,report,init,doctor,explain}.ts` (init complete; others structured stubs that load config and print "not implemented" to stderr with exit 1).
Out: pipeline logic, doctor probes, report rendering.

## Detailed Requirements

1. `src/cli/main.ts`: commander program `reaper`, version from package.json. Global options: `--config <path>`, `--cwd <path>` (process.chdir before anything), `--log-level <level>` (error|warn|info|debug, default info), `--no-color`. Construct Logger (issue 02) once; pass a `CliContext { config: ResolvedConfig; log: Logger; cwd: string }` to commands.
2. Subcommands + flags exactly per DESIGN.md §7 table:
   - `scan`: `--json <path>`, `--md <path>`, `--fail-on <band>`, `--strict-tools`, `--only <adapters>` (comma list validated against the 5 adapter names), `--filter <expr>` (parsed as `;`-separated `key=value`, keys `category|language|band`).
   - `propose`: `--max-prs <n>` (int 1..10), `--dry-run`, `--base <branch>`, `--scan <path>`.
   - `report`: `--json <path>`, `--md <path>`.
   - `init`: `--force`.
   - `doctor`: `--strict-tools`.
   - `explain <fingerprint>`: `--scan <path>`; fingerprint arg validated `/^[0-9a-f]{16}$/`.
3. Exit-code discipline: every command body is wrapped by one handler mapping `ReaperError.code` → exit code table (§7): `E_CONFIG_*`→2, `E_TOOL_MISSING_STRICT`→4, `E_PUBLISH_*`→5, `--fail-on` breach→3 (thrown as `E_FAIL_ON`), anything else→1 with stack at debug level. Success paths return 0. No `process.exit` calls outside `main.ts`.
4. Flag validation errors (bad enum value, non-int) print usage hint to stderr and exit 2.
5. `reaper init` (complete implementation):
   - Refuses if `reaper.jsonc` exists unless `--force` (exit 2, message).
   - Writes `STARTER_CONFIG` (issue 03) to `<cwd>/reaper.jsonc`.
   - Appends `.reaper/` to `.gitignore` if the file exists and lacks the entry; creates `.gitignore` with the entry if absent.
   - Prints next-steps block (run `reaper doctor`, then `reaper scan`).
6. Machine-output rule (§7): `--json -` writes JSON to stdout; all logs to stderr; add a test asserting stdout purity for `--json -` stubs (empty JSON object placeholder acceptable until 31).
7. `--help` output for every command includes one usage example line.

## Acceptance Criteria

- [ ] `reaper --help` lists 6 subcommands; each `<cmd> --help` shows its flags + example.
- [ ] `reaper init` creates `reaper.jsonc` + `.gitignore` entry; second run without `--force` exits 2; with `--force` overwrites.
- [ ] Invalid inputs exit 2 (`reaper scan --fail-on banana`, `reaper explain xyz`, `reaper propose --max-prs 0`).
- [ ] Config parse/validation failures from any command exit 2 with the ajv path message (integration test with a broken reaper.jsonc).
- [ ] Stub commands exit 1 with "not implemented: <cmd>" on stderr, nothing on stdout.
- [ ] No `process.exit` outside `main.ts` (lint rule or grep test).

## Validation

`pnpm build && node dist/cli/main.js --help`; scripted CLI tests via vitest spawning the built binary (use `node dist/cli/main.js` through ExecRunner-free plain spawn in tests).

## Dependencies

01, 02, 03.

## Non-goals

scan/propose/report/doctor/explain bodies (13, 30, 31); shell completions (v2).

## Design References

DESIGN.md §7 (contract), §6 (init content via STARTER_CONFIG), §16 (stdout purity).
