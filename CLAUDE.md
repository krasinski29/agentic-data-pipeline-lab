# CLAUDE.md

Guidance for any Claude Code / agent session working in this repository.
This file documents *decisions already made*, not aspirations — keep it
in sync with reality as the project evolves.

## What this project is

`pipeline-lab` (import name `pipeline_lab`) is a learning project: an
end-to-end batch data pipeline, built with a deliberate focus on using AI
agents (Claude Code) throughout development — not just as a data-pipeline
exercise, but as a hands-on way to learn agent-driven workflows, skills,
and tooling.

The pipeline's own architecture (ingestion, transformation, storage) has
not been designed yet. Do not invent pipeline logic or add dependencies
for it without an explicit go-ahead — that design happens in its own
phase, discussed before any code is written.

## Environment

- **Python 3.12**, pinned in `.python-version`.
- **`uv`** manages the interpreter, virtual environment, dependencies and
  lockfile. There is no `pip`/`poetry`/`pyenv` in this project — don't
  suggest or use them.
- Never edit `.venv/` or `uv.lock` by hand. Add/remove dependencies with
  `uv add <pkg>` / `uv add --dev <pkg>` / `uv remove <pkg>`, then commit
  the resulting `pyproject.toml` and `uv.lock` together.

## Everyday commands

```bash
uv sync                       # install/update the environment from uv.lock
uv run pytest                 # run tests
uv run ruff check .           # lint
uv run ruff format .          # format
uv run pre-commit run --all-files   # run all pre-commit hooks manually
uv run pipeline-lab           # run the CLI entry point
```

## Layout

```
src/pipeline_lab/   # the package — all importable source code lives here
tests/               # pytest tests, mirroring the src/ structure
```

This is a **src layout**, chosen deliberately: code is only importable
because it's installed into `.venv` (via `uv sync`), not because it sits
next to the tests. This catches packaging mistakes that a flat layout
would hide. When adding a new module under `src/pipeline_lab/`, add a
corresponding test under `tests/`.

## Quality gates

Three layers, in increasing order of strictness:

1. **Local hook** (`.pre-commit-config.yaml`, installed via
   `pre-commit install`) — runs `ruff-check --fix` and `ruff-format` on
   `git commit`. Skippable (`--no-verify`), only active if installed.
2. **CI** (`.github/workflows/ci.yml`) — runs on every push/PR against
   `main`: `uv sync --locked`, `ruff check`, `ruff format --check`,
   `pytest`. This is the check nobody can bypass.
3. **Manual review** — every PR is merged by the repo owner, not
   auto-merged, even when CI passes.

Before proposing a change as done, run `uv run ruff check .` and
`uv run pytest` yourself and report the actual output — don't assume.

## Git workflow

- **Never commit directly to `main`.** Every change goes through a
  branch and a PR, even small config changes.
- Branch naming: `<type>/<short-description>`, e.g.
  `chore/quality-tooling`, `feat/ingest-csv`.
- **Commit messages and PR titles follow [Conventional Commits](https://www.conventionalcommits.org/):**
  `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `ci:`, `refactor:` — a
  type prefix, present tense, no trailing period. This is enforced by
  convention, not by a commit-lint tool (yet).
- **Merges are always manual**, done by the repo owner after reviewing
  the PR — even when CI is green. Do not merge PRs automatically.

## What NOT to do without asking

- Don't add dependencies, especially cloud SDKs (AWS/GCP/Azure) —
  the pipeline's runtime target hasn't been decided yet, and the design
  intentionally stays portable until it is.
- Don't restructure `src/pipeline_lab/` or introduce pipeline
  architecture (ingestion/transform/load modules) — that's a separate,
  explicitly-scoped phase.
- Don't change CI, pre-commit, or dependency-management tooling without
  walking through the change first — this repo exists partly to teach
  *why* each piece is there, not just to have it working.
