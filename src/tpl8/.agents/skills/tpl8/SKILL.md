---
name: tpl8
description: A greeter CLI (hello world) — greets someone by name. Use when asked to run tpl8, greet someone, or adjust the greeting.
---

<!--
SPDX-License-Identifier: 0BSD
Copyright (c) 2026 tpl8 creators and contributors
-->

# tpl8

A greeter CLI.

```
$ uv run tpl8 --name Broman --formal
Good day from tpl8 to Broman!
```

Two invocation paths: `uv run tpl8 ...` and `uv run python -m tpl8 ...`.
Run `uv run tpl8 --help` to see the options (`--name`, `--formal`).
