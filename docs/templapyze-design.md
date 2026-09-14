# Design proposal: `templapyze`

**One-liner.** `templapyze <dir>` bootstraps a new Python project in `<dir>`
from a *template* — where a template is a real, working, publishable project
(like tpl8), not a directory of `{{placeholders}}`. It generalizes testafize:
instead of "Makefile + config snippet + `curl` from raw.githubusercontent",
the whole project is the template and the command is an actual executable.

```
$ templapyze tpL8
[command output]
$ tree tpL8/
tpL8/
├── AGENTS.md
├── Makefile
├── pyproject.toml
├── README.md
├── scripts
│   └── test-all-versions.sh
├── src
│   └── tpl8
│       ├── __init__.py
│       ├── __main__.py
│       └── py.typed
├── tests
│   └── test_greet.py
└── uv.lock
```

## Core decision: a template is a distribution

- A template is an **installed Python dist** (or a path / git checkout of
  one). The template package *is* the starting project — its `src/<pkg>/`
  ships the example code, and its wheel ships the skill at
  `src/<pkg>/.agents/skills/<pkg>/SKILL.md`, so the skill is easy to find in
  the installed template.
- Non-package files the new project inherits (Makefile, AGENTS.md,
  `scripts/`, `.github/`, `.gitignore`, `.python-version`) live next to the
  package in the template repo and are listed in a manifest (below).
- Payoff: the template stays a *live* project — it keeps passing its own
  `make test`, gets PRs and releases, and `templapyze` consumes it like any
  other dependency. No frozen scaffold, no jinja sprawl.

## The manifest (the crux)

Blind textual replacement of the template's package name is how you ship
bugs: `tpl8` is a short token that could appear inside words, URLs, emails —
and the replacement list is exactly what you want to audit. So the template
declares its personalization points in its own `pyproject.toml`:

```toml
[tool.templapyze]
package = "tpl8"        # canonical self-name; becomes the new package name
files = ["Makefile", "AGENTS.md", ".gitignore", ".python-version",
         "scripts/test-all-versions.sh", ".github/workflows/ci.yml"]
```

Rename rules:

1. **Path renames:** `src/<pkg>/` → `src/<newpkg>/`, including the nested
   `.agents/skills/<pkg>/` → `<newpkg>/` (this is why the skill dir is named
   after the package).
2. **Textual replacement:** the token `<pkg>` (word-boundary match) is
   rewritten in the manifest-listed files plus `pyproject.toml` and
   `src/<newpkg>/`. Nothing else is touched. `uv.lock` is never copied
   (see pipeline).
3. **Personalization fields** (description, author) come from flags or git
   config, not from replacement.
4. `templapyze --plan` prints the rename table before anything is written —
   the auditability hook.

Fallback when a template has no manifest: convention (self-name = the dist's
own PEP 503 name, fixed file list). Works, but templates should carry the
manifest.

## Name derivation

The argument is a **directory** name; the package name is its PEP 503
normalization (`lower()`, runs of `[-_.]` → `-`) — the same rule uv applies,
which is why `tpL8/` outside contains `src/tpl8/` inside. Print a note on
mismatch (`directory 'tpL8' → package 'tpl8'`); `--name` overrides
explicitly.

## Pipeline

1. **Safety:** target must not exist or be empty; refuse otherwise
   (`--force` to override). Never modifies existing files.
2. **Resolve template:** bundled default → `--from <path>` →
   `--from git+<url>` (clone to temp).
3. **Plan:** compute the rename table; stop here on `--plan`.
4. **Copy + rename** into the target (the tree above, minus artifacts).
5. **Never copied:** `.git`, `.venv*`, all caches (`__pycache__`,
   `.ruff_cache`, `.pytest_cache`, `.mypy_cache`, `.uv`), `dist/`, and
   `uv.lock` — the lock references the *old* project's own dist, so it is
   regenerated in step 6.
6. **Init:** write the new `pyproject.toml` (name, description, author,
   provenance) → `uv sync` (creates `.venv`, writes the fresh lock, installs
   dev tools + deps).
7. **Gate:** `make check` + `make test`. Because the template's example code
   *and its tests* are renamed in, a fresh project is green immediately — no
   "Error 5, no tests ran" limbo. Gate failure aborts with the directory
   left in place and a clear message.
8. **Git:** `git init` + first commit of the verified-green tree
   (`--no-commit` to skip). Commit message by convention:
   `Bootstrap <name>: <template> v<version>`. No git identity configured →
   fail *at this step*, after the project exists and is green, with the
   exact fix (`git config user.name/user.email`) rather than aborting the
   whole run.
9. **Report:** tree, the normalization note, next steps (`uv run <pkg>`,
   `make test`, `make test-all-versions`).

## CLI surface (minimal)

```
templapyze <dir> [--from SRC] [--name N] [--description D]
                [--author "Name <email>"] [--python X.Y]
                [--no-commit] [--force] [--plan]
```

Defaults: bundled template, name from directory, author from git config,
python from the current uv interpreter (written to `.python-version`).

Dogfooding: `templapyze` itself is a small typer + pydantic CLI built from
the tpl8 template — the template's first offspring is its own tool.

## Decisions (closed)

1. **Bundled default template.** `templapyze` is a personal opinionated
   template, not a generic migration tool — the greeter ships inside the
   `templapyze` dist. One subtlety: half the template (Makefile, AGENTS.md,
   `scripts/`, `.github/`) lives *outside* the package, so the dist
   **vendors the template as package data** — an exact, hash-pinned snapshot
   of a tpl8 release. Implemented as a **deterministic tar archive**
   (`templapyze/templates/tpl8-v0.1.0.tar` + `.sha256` sidecar), not a tree:
   the vendored template is *data*, and its Python sources must never be
   scanned by the static tools (a vendored tree breaks refurb/mypy's src
   layout resolution and would need per-tool excludes). Consequences:
   - The fantasy command works offline and deterministically: same
     templapyze version → same generated tree.
   - templapyze releases are coupled to the template snapshot; bumping the
     greeter template is a templapyze release. That coupling *is* the
     versioning story for a personal tool.
   - `--from` stays for other templates or a newer tpl8, but the default
     needs no network and no extra install.
   - The snapshot includes the template's `[tool.templapyze]` manifest, so
     the rename path is identical for bundled and external templates.

2. **First commit by default.** The pipeline order guarantees it: gate →
   `git init` → commit. `--no-commit` leaves the tree as-is, uncommitted.
   (See pipeline step 8 for the message convention and the no-git-identity
   edge case.)

3. **Provenance.** Stamped machine-readably, not just as prose:
   - Generated `pyproject.toml` gets `[tool.templapyze]` with
     `origin = "tpl8"` and `version = "0.1.0"` — same table name the
     *template* manifest uses, so a future updater can distinguish
     "templapyze-generated project" from "hand-rolled project that looks
     similar" by presence of the table, and knows what to diff against.
   - README gets the human line: `Generated by templapyze from tpl8 v0.1.0`.

## What it is not (v1)

- **Not an updater** — no "bump my project to template v2"
  (cookiecutter-`--update`-class problem; revisit with the provenance data
  from decision 3).
- **Not a configurator** — no interactive wizard; flags + sensible defaults.
- **Not a generic scaffold** — no feature flags. A different opinionated
  project is a different template dist.

## Acceptance tests

1. `templapyze tpL8` in an empty dir → the tree above; `uv run tpl8`
   greets; `make test` green; built wheel contains
   `tpl8/.agents/skills/tpl8/SKILL.md`.
2. `templapyze My.Cool_CLI` → package `my-cool-cli`, warning printed, and
   `rg -w tpl8` over the tree is **empty** — the rename-correctness test.
3. Non-empty target → clean refusal, nonzero exit, nothing written.
4. `--plan` → prints the rename table, writes nothing.
5. Template without manifest → convention fallback works.
6. The template itself keeps passing its own `make test` (live project
   invariant).

## Load-bearing pieces

- **Dist-as-template** — keeps the template alive and the skill findable.
- **Manifest-driven rename** — explicit and testable; acceptance test 2
  exists precisely because this is the part that breaks.
- **Gate-before-commit ordering** — only verified-green trees become first
  commits.
