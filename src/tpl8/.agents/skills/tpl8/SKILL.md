---
name: tpl8
description: Conventions for the tpl8 template and projects bootstrapped from it (typer + pydantic CLI, uv, make check/test pipeline). Use when modifying this template or a tpl8-based project.
---

# tpl8

The repository-root `AGENTS.md` is the full reference — read it first.

Quick orientation:

- CLI split: typer parses args and owns exit codes; a pydantic model validates
  and renders; the command is thin glue (`ValidationError` → stderr +
  `typer.Exit(code=1)`).
- Entry point is `main()` in `src/<pkg>/__main__.py`; `python -m <pkg>` must
  keep working alongside the console script.
- Gate: `make test` (static pipeline + pytest). Changes to dependencies or
  `requires-python` also need `make test-all-versions`.
- Keep three things in sync: `requires-python`, the version list in
  `scripts/test-all-versions.sh`, and the CI matrix in `.github/workflows/ci.yml`.
- Docstrings are required on public definitions (interrogate `fail-under = 90`).
- Never run a bare `uv run --python <older-version>`: it recreates the dev
  `.venv`. Use `UV_PROJECT_ENVIRONMENT=".venv-<ver>"` instead.
