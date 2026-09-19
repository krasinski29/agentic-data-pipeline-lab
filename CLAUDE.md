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

## Layout

```
src/pipeline_lab/   # the package — all importable source code lives here
tests/               # pytest tests, mirroring the src/ structure
docs/decisions/      # ADRs — one frozen record per architecture decision
docs/erd-*.md        # ERD derived from the ADRs — the ADR wins if they diverge
```

**src layout, chosen deliberately**: code is only importable because
it's installed into `.venv` (via `uv sync`), not because it sits next to
the tests. This catches packaging mistakes a flat layout would hide.
When adding a module under `src/pipeline_lab/`, add a corresponding test
under `tests/`.

## Workflow — use the skills, don't improvise the procedure

- **Before reporting any change as done, passing, or ready** — run
  `/quality-check`. Never claim tests pass or lint is clean without
  having just run it.
- **To propose any change** — run `/open-pr`. It encodes this repo's
  branch naming, Conventional Commits format, PR structure, and the
  rule that **merges are always manual**, done by the repo owner, never
  by the agent, even when CI is green.
- **To record an architecture decision** — run `/record-decision`. It
  encodes where each piece goes (Linear document for the reasoning,
  frozen ADR under `docs/decisions/`, issue description for the
  summary), the rule that **an accepted ADR is superseded, never
  edited**, and the duty to propagate consequences to the issues that
  inherit them.
- **Never commit directly to `main`.** No exception, including for
  small config/tooling changes.

If a skill's procedure and this file ever disagree, the skill is
out of date — fix the skill, don't just follow this file instead.

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
