---
name: quality-check
description: Run this repo's full local quality gate (ruff lint, ruff format check, pytest) and report the actual results. Use before claiming any change is "done", "passing", or "ready for PR" — never state that without having just run this.
license: CC-BY-4.0
metadata:
  author: krasinski29
  version: '1.0.0'
---

# Quality check

Run these three commands, in order, and report their real output — do
not summarize as "passed" without showing what actually ran. Stop and
report the failure if any step fails; do not continue to the next step.

```bash
uv run ruff check .
uv run ruff format --check .
uv run pytest
```

## Interpreting results

- `ruff check` failures: fix what's auto-fixable with
  `uv run ruff check --fix .`, then re-run to confirm clean. Anything
  not auto-fixable needs a manual code change — do not silence with
  `# noqa` without asking, since that hides the underlying issue rather
  than fixing it.
- `ruff format --check` failures: run `uv run ruff format .` to apply
  formatting, then re-run `--check` to confirm.
- `pytest` failures: report the failing test name and the actual
  assertion error. Do not guess at a fix without understanding why the
  test failed.

## When this matters

This mirrors exactly what CI (`.github/workflows/ci.yml`) runs. Running
it locally first means a PR shouldn't surprise you with a red check —
if this skill passes, CI should too (same commands, same lockfile via
`uv.lock`).
