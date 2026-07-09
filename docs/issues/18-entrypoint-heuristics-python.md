# Title

Python entry-point & public-surface heuristics (S6/S8)

## Summary

Implement the Python entry-point pack: framework/tooling-invoked callables and module conventions (S6, −0.60) and package public surface (S8, −0.30 for libraries), per DESIGN.md §10.5.

## Context

vulture's classic false positives are framework-invoked: Django views/management commands, celery tasks, pytest fixtures, CLI entry points declared in pyproject. These heuristics are the compensation layer that makes vulture findings usable (research survey §2).

## Scope

In: `src/signals/entrypoints/python.ts` + tests (pack contract from 17).
Out: TS/Go packs, dynamic-usage patterns (16), executing any Python.

## Detailed Requirements

1. Rule table `PY_ENTRYPOINT_RULES` (ids in evidence). S6 fires when:
   - **pyproject declarations:** symbol/module referenced by `[project.scripts]`, `[project.gui-scripts]`, `[project.entry-points.*]` (TOML parse: minimal reader — reuse/extract the line-based TOML reader planned for 26 into `src/signals/toml-lite.ts` created HERE, consumed by 26), or poetry equivalents `[tool.poetry.scripts]`. Match: entry value `pkg.mod:func` ⇒ the module file and the function symbol.
   - **Decorator conventions** (textual, comment-stripped via lex.ts): finding is a function/method whose declaration line-region (5 lines above the def at `startLine`) contains a decorator matching: `@app.route, @router.(get|post|put|delete|patch|websocket), @api_view, @celery_app.task, @task, @shared_task, @pytest.fixture, @click.command, @click.group, @app.command, @cli.command, @receiver, @admin.register, @register.filter, @app.(on_event|middleware), @validator, @field_validator`.
   - **File conventions:** `conftest.py` (all symbols), `settings.py`/`settings/**` in Django-marked repos (a file importing `django` exists), `manage.py`, `wsgi.py`, `asgi.py`, `migrations/**` (also protected by default config), `management/commands/**` (Command class), `apps.py` (AppConfig), `tasks.py` when celery imported in repo, `__main__.py`.
   - **Dunder conventions:** symbols named `__*__` (vulture often flags custom dunders used by protocols).
   - **unittest/pytest conventions:** symbols `setUp, tearDown, setUpClass, tearDownClass, setup_method, teardown_method` on classes in test-class files (these come through when vulture scans tests).
2. S8 (repoKind=library): `unused-symbol`/`unused-export`-equivalent findings whose name is listed in the package root `__init__.py`'s `__all__`, or imported by any `__init__.py` in an ancestor package directory (textual `from .x import name` / `from pkg.x import name` parse) ⇒ S8 with evidence `init-reexport:<path>`.
3. All parsing is textual/AST-free and comment-stripped; malformed pyproject ⇒ warn + skip that source.
4. Caps: decorator scan reads only files containing findings (via index memoization); evidence ≤ 3.
5. Tests: rule matrix — each rule id firing + non-firing (e.g. `@app.route` on the finding vs. on an unrelated function; entry-point `pkg.mod:func` matching exactly that func and not same-named func elsewhere); `__all__` and init-reexport S8 with repoKind gating; dunder rule; toml-lite reader table tests (arrays, inline tables minimal support, comments preserved-ignored).

## Acceptance Criteria

- [ ] Every rule id covered both ways; decorator region logic verified at region boundaries (5-line window).
- [ ] `[project.scripts] mycli = "pkg.cli:main"` protects exactly `pkg/cli.py:main` (test).
- [ ] S8 init-reexport + `__all__` paths verified; application repos get no S8.
- [ ] The 33-fixture traps (celery task, pytest fixture, `getattr` registry target ALSO covered by 15/16) map to at least one firing rule here or in 15/16 — traceability table included in the PR description.

## Validation

`pnpm test src/signals/entrypoints/python* src/signals/toml-lite*`.

## Dependencies

14, 17 (pack contract + shared config-glob helper), 15/16 lexer (`lex.ts`).

## Non-goals

Executing Python, import-graph resolution, Django URL-conf resolution beyond decorators/globs (v2), notebook support.

## Design References

DESIGN.md §10.5 S6/S8, §18.1 traps; research survey §2 (vulture blind spots); ADR-003.
