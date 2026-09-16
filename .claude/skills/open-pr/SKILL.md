---
name: open-pr
description: Open a pull request in this repo following its established convention — branch naming, Conventional Commits, structured PR body, and manual-merge-only. Use whenever changes are ready to be committed and proposed, in this project.
license: CC-BY-4.0
metadata:
  author: krasinski29
  version: '1.0.0'
---

# Open a PR

This repo never commits directly to `main`. Every change — including
small config/tooling changes — goes through this flow.

## 1. Run `/quality-check` first

If the change touches Python code, run the quality-check skill before
committing. Don't propose a PR with known-failing checks.

## 2. Create a branch

```bash
git checkout -b <type>/<short-description>
```

`<type>` matches the Conventional Commits type below (`feat`, `fix`,
`chore`, `docs`, `test`, `ci`, `refactor`). Keep the description a few
kebab-case words, e.g. `chore/quality-tooling`, `docs/claude-md`.

## 3. Stage and commit

Stage only the files that belong to this change — check `git status`
first rather than reflexively using `git add -A`.

Commit message follows [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>: <present-tense summary, no trailing period>

<body explaining what changed and, more importantly, why —
not just a restatement of the diff>
```

## 4. Push and open the PR

```bash
git push -u origin <branch-name>
gh pr create --base main --title "<same as commit summary>" --body "..."
```

PR body structure — use these headings:

- **What** — what changed, file by file if it's not obvious
- **Why** — the reasoning; this is the part a diff alone doesn't show
- **Verification** — the actual commands run and their actual output
  (from step 1), not an assumption that it works

## 5. Wait for manual merge — never merge automatically

Report the PR URL and stop. **Do not merge the PR yourself, even if CI
is green.** The repo owner reviews and merges manually — this is a
deliberate project decision, not a missing capability.

## 6. After the user confirms the merge

Verify before syncing — don't take "mergeado" at face value:

```bash
gh pr view <number> --json state,mergedAt
```

Then sync and clean up:

```bash
git checkout main
git pull
git branch -d <branch-name>
```
