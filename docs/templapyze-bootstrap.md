<!--
SPDX-License-Identifier: 0BSD
Copyright (c) 2026 tpl8 creators and contributors
-->

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

## Phase B — manual templapyze run into `~/templapyze` (done)

**Write down every step and decision as you go** — that log becomes the
command's spec and integration-test fixture. Log: `~/templapyze/docs/bootstrap-log.md`.

- [x] `mkdir ~/templapyze`; copy the template tree from tagged `v0.1.0`
      (manifest `files` + `src/tpl8/`), excluding `.git`, `.venv*`, caches
      (`__pycache__`, `.ruff_cache`, `.pytest_cache`, `.mypy_cache`, `.uv`),
      `dist/`, `uv.lock`, and `docs/` (design docs are tpl8-specific).
- [x] Path renames: `src/tpl8` → `src/templapyze`;
      `src/templapyze/.agents/skills/tpl8` → `src/templapyze/.agents/skills/templapyze`.
- [x] Token rename `tpl8` → `templapyze` (word-boundary) in: `pyproject.toml`,
      `Makefile`, `AGENTS.md`, `SKILL.md`, `ci.yml`, `test-all-versions.sh`,
      `src/templapyze/*.py`.
- [x] Personalize: description/author; entry point
      `templapyze = "templapyze.__main__:main"`; **replace** the template's
      `[tool.templapyze]` manifest table with the provenance block
      (`origin = "tpl8"`, `version = "0.1.0"`). Same table name, different
      fields — the generator must *replace*, never copy.
- [x] `uv sync` → gate: `make check` + `make test`. The renamed *greeter* must
      work (`uv run templapyze` says hello) — this verifies the rename before
      any real code exists.
- [x] `rg -w tpl8` — expect exactly **two** hits in project files, both
      intentional provenance lines: pyproject `origin = "tpl8"` and the
      README stamp. (The command's rename-correctness test must exclude
      provenance locations.)
- [x] `git init` + first commit:
      `Bootstrap templapyze: tpl8 v0.1.0 (manual templapyze run)`.
- [x] Keep the golden tree: tag it `bootstrap-v0` — it is the diff target for
      Phase D.

## Phase C — implement the tool in `~/templapyze` (`make test` green throughout) (done)

Deviations and gotchas: `~/templapyze/docs/bootstrap-log.md` (Phase C section).

- [x] Vendor the snapshot: pinned tpl8 tag →
      `src/templapyze/templates/tpl8-v<version>.tar` + `.sha256` sidecar
      (**tar archive, not a tree** — the static tools must not scan vendored
      code); the wheel ships both (verified). Re-vendored as `v0.2.0` after
      the Makefile uv<0.12 security-scan fix.
- [x] Replace the greeter with the real CLI: typer surface
      (`dir`, `--plan`, `--force`, `--no-commit`, `--from`, `--name`,
      `--description`, `--author`, `--python`), pydantic models
      (`Author`, `TemplateSpec`, `RenamePlan`).
- [x] Implement pipeline steps 1–9 from the design doc (gate = `make test`,
      which already depends on `check`).
- [x] Tests: name normalization, rename-table planning, snapshot integrity,
      safety refusals, and an `integration`-marked end-to-end test (generate
      → gate → commit → token audit) run via `make test-integration`.
- [x] Rewrite the greeter content (SKILL.md, README, AGENTS.md) for the
      real tool.

## Phase D — recursive verification (the payoff) (done)

- [x] Run the finished command: `cd /tmp && templapyze templapyze` (same
      target name as the manual run, so the diff is meaningful) → fresh
      project, gate green in-pipeline, first commit made.
- [x] `diff -r` against the golden tree (`bootstrap-v0`) — actual deltas:
      description (template placeholder, known follow-up), provenance comment
      wording, manual-only `docs/bootstrap-log.md`, untracked artifacts.
      **`uv.lock` byte-identical**; everything else byte-identical.
- [x] `make test` green in the generated project (6 passed).
      (Design acceptance test 1, run for real.)
