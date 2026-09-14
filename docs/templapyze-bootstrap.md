# Bootstrap checklist: building `templapyze` from the tpl8 template

Handoff doc. The `templapyze` command does not exist yet, so the first
bootstrap into `~/templapyze` is a **manual templapyze run** — and that manual
run is the reference implementation: every step taken by hand becomes a spec
item and a test fixture for the real command. See `templapyze-design.md` for
the design this checklist operationalizes.

## Phase A — make tpL8 a valid template (done in this repo)

- [x] Add the `[tool.templapyze]` manifest to `pyproject.toml`
      (`package = "tpl8"`, `files = [...]`); `make check` green.
- [x] Tag the snapshot source: `v0.1.0` (matches `version = "0.1.0"`).
- [x] Commit both.

## Phase B — manual templapyze run into `~/templapyze`

Do this in a fresh session. **Write down every step and decision as you go** —
that log becomes the command's spec and integration-test fixture.

- [ ] `mkdir ~/templapyze`; copy the template tree from tagged `v0.1.0`
      (manifest `files` + `src/tpl8/`), excluding `.git`, `.venv*`, caches
      (`__pycache__`, `.ruff_cache`, `.pytest_cache`, `.mypy_cache`, `.uv`),
      `dist/`, `uv.lock`, and `docs/` (design docs are tpl8-specific).
- [ ] Path renames: `src/tpl8` → `src/templapyze`;
      `src/templapyze/.agents/skills/tpl8` → `src/templapyze/.agents/skills/templapyze`.
- [ ] Token rename `tpl8` → `templapyze` (word-boundary) in: `pyproject.toml`,
      `Makefile`, `AGENTS.md`, `SKILL.md`, `ci.yml`, `test-all-versions.sh`,
      `src/templapyze/*.py`.
- [ ] Personalize: description/author; entry point
      `templapyze = "templapyze.__main__:main"`; **replace** the template's
      `[tool.templapyze]` manifest table with the provenance block
      (`origin = "tpl8"`, `version = "0.1.0"`). Same table name, different
      fields — the generator must *replace*, never copy.
- [ ] `uv sync` → gate: `make check` + `make test`. The renamed *greeter* must
      work (`uv run templapyze` says hello) — this verifies the rename before
      any real code exists.
- [ ] `rg -w tpl8` — expect exactly **one** hit: the provenance
      `origin = "tpl8"`. (The command's rename-correctness test must exclude
      the provenance line.)
- [ ] `git init` + first commit:
      `Bootstrap templapyze: tpl8 v0.1.0 (manual templapyze run)`.
- [ ] Keep the golden tree: tag it `bootstrap-v0` — it is the diff target for
      Phase D.

## Phase C — implement the tool in `~/templapyze` (`make test` green throughout)

- [ ] Vendor the snapshot: pinned tpl8 `v0.1.0` tree →
      `src/templapyze/templates/tpl8/` + sha256 pin; verify the wheel ships it
      (`uv build`, list contents).
- [ ] Replace the greeter with the real CLI: typer surface
      (`dir`, `--plan`, `--force`, `--no-commit`, `--from`, `--name`,
      `--description`, `--author`, `--python`), pydantic model for the rename
      plan.
- [ ] Implement pipeline steps 1–9 from the design doc.
- [ ] Tests: name normalization, rename-table planning, tmp-dir integration
      (run command → `rg -w tpl8` minus provenance → generated project's
      `make test`).

Note: after Phase B, `templapyze`'s SKILL.md/README still describe a *greeter*
— mechanical rename ≠ content. Rewriting that content for the real tool is
part of this phase; the project's own AGENTS.md is what guides that work (the
template's first real use).

## Phase D — recursive verification (the payoff)

- [ ] Run the finished command: `cd /tmp && <templapyze> demo` → fresh greeter
      project.
- [ ] `diff -r` against the golden tree (`bootstrap-v0`) — expected deltas:
      provenance stamp, description, `uv.lock` only.
- [ ] `make test` green in the generated project.
      (Design acceptance test 1, run for real.)
