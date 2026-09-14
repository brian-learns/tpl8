# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this is

A template Python project: a minimal CLI (`tpl8`) built with **typer** (argument
parsing, help, exit codes) and **pydantic** (validation, domain logic), managed
with **uv**. The static-test pipeline and Makefile conventions are the point of
the template as much as the example code — keep them working.

- Package name: `tpl8` (uv derives it from the directory name; it is
  lowercase-normalized — the directory may not match).
- Supported Python: `>=3.12` (see `requires-python`). Local dev pins 3.14 via
  `.python-version`; CI and `make test-all-versions` cover 3.12/3.13/3.14.

## Commands

| Command | Purpose |
| --- | --- |
| `make check` | Full static pipeline: lockfile check, ruff, bandit, vulture, refurb, ty, interrogate, `uv audit` (incl. malware check). All must pass. |
| `make test` | `make check` + `pytest -v --durations=5`. |
| `make test-all-versions` | `pytest` on every supported Python version (per-version `.venv-<ver>` envs, dev `.venv` for 3.14). |
| `make format` | `ruff format` + `ruff check --fix` on `src/`. |
| `make init` | Bootstrap a new project from this template (dep check, `uv init --package .`, add dev tools). |
| `uv run tpl8` | Run the CLI. `uv run python -m tpl8` works too (both use `main()` in `src/tpl8/__main__.py`). |

Run `make test` before reporting any change done. For changes touching
`pyproject.toml` dependencies or `requires-python`, also run
`make test-all-versions` (and update the CI matrix and the script's version
list — see Coupling notes below).

## Layout

```
src/tpl8/__init__.py       package docstring only
src/tpl8/__main__.py       typer app, pydantic model, main() entry point
src/tpl8/py.typed          PEP 561 marker — must stay in the built wheel
src/tpl8/.agents/skills/tpl8/SKILL.md
                           minimal agent skill, points back to this file
tests/test_greet.py        CliRunner tests for the CLI + direct model tests
scripts/test-all-versions.sh
Makefile                   the pipeline; `check` is the gate
.github/workflows/ci.yml   CI matrix, mirrors the local version list
```

## Conventions

- **CLI design**: typer owns the interface (options, help, exit codes);
  pydantic owns the domain (validation, rendering). Commands are thin glue:
  build the model, catch `ValidationError` → message to stderr +
  `typer.Exit(code=1)`, else `typer.echo(model.render())`.
- **pydantic**: use `mode="before"` validators when normalization must happen
  *before* field constraints (e.g. `strip()` before `min_length`). Plain
  functions work as validators (pydantic v2); they avoid vulture flagging an
  unused `cls` — but ruff N805 misfires on them in class scope, so keep the
  `# noqa: N805` with a short reason.
- **Tests**: CLI behavior via `typer.testing.CliRunner` (assert `exit_code`
  and `output`/`stderr`); model behavior directly; `pytest.raises` for
  expected `ValidationError`s.
- **Docstrings**: interrogate enforces `fail-under = 90` on `src/` — every
  public function, method, and class needs one.
- **Style**: ruff (line length 120; rules E, F, B, S, I, N, RUF, ANN —
  type hints are mandatory; ANN002/ANN003 are ignored).
  `raise ... from err` inside `except` blocks (B904).

## Coupling notes (keep these in sync)

- `requires-python` in `pyproject.toml` ⇄ version list in
  `scripts/test-all-versions.sh` ⇄ `python-version` matrix in
  `.github/workflows/ci.yml`. Bumping the supported range means updating all
  three.
- The last version in the script runs in the dev `.venv` and must match
  `.python-version`; the others get `UV_PROJECT_ENVIRONMENT=".venv-<ver>"`
  envs. Never run a bare `uv run --python <older>` — it would recreate the
  dev `.venv` with that interpreter.
- Dependency resolution uses `exclude-newer = "7 days"` (see `[tool.uv]`);
  `uv lock --check` in `make check` enforces it. CI installs with
  `uv sync --frozen` against the committed lockfile.

## Gotchas

- `uv` >= 0.12 is required (checked by `make checkdeps`); the audit and
  malware-check flags are preview features.
- Inside a project nested in an outer git repo, `uv init` won't write a local
  `.git`/`.gitignore`; run `GIT_CEILING_DIRECTORIES=$HOME make init` (must be
  an ancestor of the project dir).
- `make test` failing with pytest Error 5 "no tests ran" is expected on a
  fresh init, before any tests exist.
- Venvs (`.venv`, `.venv-*`) and tool caches are gitignored — never commit
  them; `make clean` removes the caches.
