<!--
SPDX-License-Identifier: 0BSD
Copyright (c) 2026 tpl8 creators and contributors
-->

# tpl8: template-change workflow

tpl8 is consumed by templapyze as a pinned, hash-verified snapshot
(`src/templapyze/templates/tpl8-vX.Y.Z.tar` + `.tar.sha256`). Any change to
tpl8 — including pipeline fixes in the Makefile, which is duplicated in
templapyze — requires a new tag **and** a re-vendor in templapyze.
Documentation-only changes (`docs/`) are the sole exception: they ship in
neither the snapshot nor the generated projects, so no re-vendor is needed.

## 1. Make the change in tpl8

- Edit the code.
- If the Makefile pipeline changed, apply the identical change to
  `~/templapyze/Makefile` (they are duplicated by design).
- `make test` (the gate; it depends on `check`).
- If you touched the template manifest (`[tool.templapyze]`) or anything the
  generator renders, re-run the integration test on the templapyze side
  (step 4) — the rename-correctness test asserts exactly 2 `tpl8` provenance
  tokens in generated output.

## 2. Version bump, commit, tag, push

- Bump `version` in `pyproject.toml` (and the manifest, if it pins one).
- Commit with `Bump to vX.Y.Z` (or a descriptive message).
- Tag and push **both** branch and tag:

  ```sh
  git tag vX.Y.Z
  git push origin main vX.Y.Z
  ```

  The tag is what templapyze's `resolve_template` can fetch
  (`git+https://github.com/brian-learns/tpl8@vX.Y.Z`), so CI is green and the
  tag is pushed before you re-vendor.

## 3. Re-vendor in templapyze

From the templapyze checkout, build a deterministic snapshot of the tagged
tree, minus `docs/` and `uv.lock` (docs ship in neither the tar nor generated
projects; `uv.lock` is regenerated per project):

```sh
git clone --depth 1 --branch vX.Y.Z https://github.com/brian-learns/tpl8 /tmp/tpl8
cd /tmp/tpl8
find . -type f ! -path './docs/*' ! -name uv.lock | sed 's|^\./||' | sort \
  | tar --format=ustar --owner=0 --group=0 --numeric-owner \
    -cf ~/templapyze/src/templapyze/templates/tpl8-vX.Y.Z.tar -T -
cd ~/templapyze
sha256sum src/templapyze/templates/tpl8-vX.Y.Z.tar \
  | sed 's|.*templates/|templates/|' \
  | awk '{print $1"  "$2}' > src/templapyze/templates/tpl8-vX.Y.Z.tar.sha256
```

The ustar flags and sorted file list make the tar byte-reproducible;
`verify_snapshot` in `plan.py` enforces the sha256 before any generation.

Then in `~/templapyze`:

- `src/templapyze/plan.py`: point `BUNDLED_ARCHIVE` at the new tar.
- `tests/`: update the version assertion (the bundled-snapshot test pins the
  version in the manifest).
- `AGENTS.md`: update the tar reference if it names the file.
- If the Makefile pipeline changed, it was already mirrored in step 1.

## 4. Verify, commit, push

```sh
cd ~/templapyze
make check
make test
make test-integration   # end-to-end: generated project must pass its own make test
```

Commit (e.g. `Re-vendor tpl8 vX.Y.Z`), push, and let CI run `make test` on the
3.12/3.13/3.14 matrix.

## Notes

- Never edit the vendored tar in place; always rebuild it from the tag.
- `uvx --from git+https://github.com/brian-learns/templapyze templapyze`
  caches: users need `uvx --refresh ...` to pick up a new bundled snapshot.
- Recursion check: templapyze's own `pyproject.toml` carries a
  `[tool.templapyze]` provenance stamp, not a template manifest, so it is not
  currently a template itself.
