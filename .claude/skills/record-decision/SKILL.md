---
name: record-decision
description: Record an architecture decision in this project across its three destinations — the Linear document (reasoning, evolves), a numbered ADR in docs/decisions/ (frozen, superseded rather than edited), and the issue description — then propagate the consequences to dependent issues. Use whenever an issue labeled architecture-decision is being worked, whenever a decision is reached that future work will depend on, and whenever the user says something like "vamos trabalhar no KRA-N", "registra a decisão", "vira ADR", "supersede o ADR N" or "ficou decidido". Also use before closing any issue whose completion criterion is a decision being registered. Do NOT use for implementation choices the code already documents, or to open a PR for an ordinary change — use open-pr directly for that.
license: CC-BY-4.0
metadata:
  author: krasinski29
  version: '1.0.0'
---

# Record an architecture decision

This project decides late and on purpose, so its decisions carry more weight than its code. A decision that only exists in a chat log is a decision that will be re-litigated from scratch in three months.

Records are written in Portuguese, matching the Linear workspace.

## Three destinations, three different jobs

Getting this wrong is the most common mistake — they are not copies of each other.

| Destino | Papel | Evolui? |
| -- | -- | -- |
| Documento no Linear | O raciocínio completo, estado atual | Sim, sempre |
| ADR em `docs/decisions/` | Registro congelado de **uma** decisão | Não — supersede-se |
| Descrição da issue | Resumo: Contexto / Decisão / Consequências | Congela com a issue |

The split exists because the two readers are different. Someone asking "what do we do today?" reads the Linear document. Someone asking "why did we end up here?" reads the ADR chain — and that chain is only useful if the abandoned reasoning is still in it.

## Two entry points

1. **An issue labeled `architecture-decision`** — the common case. Run the full procedure below.
2. **A decision that surfaces mid-conversation, with no issue of its own** — equally valid, and easy to miss. ADR 0002 was born this way, from a question asked while reviewing something else. Skip steps 1 and 6 and record it the same way.

## Procedure

**1. Move the issue to In Progress when work starts.** Not just Done at the end — the board should reflect reality while the work happens.

```
save_issue(id: "KRA-N", state: "In Progress")
```

**2. Gather context before proposing.** The issue itself, the domain definition document, previous ADRs, `CLAUDE.md`, and the issues that will inherit consequences. A decision made without reading the ladder above it tends to break something two degraus later.

**3. Propose with a recommendation, then ask.** The open questions on an `architecture-decision` issue belong to the repo owner — they are the deliverable, not an obstacle. Bring concrete numbers and a recommended answer with its reasoning; a menu of options with no opinion pushes the analysis back onto the person who asked for it. Use AskUserQuestion for the forks that genuinely change the outcome, and settle the rest with a sensible default. Then let them decide.

**4. Write the record** into the three destinations above:

```
save_document(title: "...", project: "agentic-data-pipeline-lab", icon: ":emoji_code:", content: "...")
save_issue(id: "KRA-N", patch: [{op: "replace", old_string: "...", new_string: "..."}])
save_issue(id: "KRA-N", links: [{url: "<pr url>", title: "PR #N — <title>"}])
```

Prefer `patch` over resending a whole description or document.

**5. Propagate the consequences.** This is the step that gets skipped. A consequence that lands on another issue does not exist until it is written *in that issue*. If the decision constrains how raw data lands, say so in the issue that chooses the format — its author will not read this ADR before deciding.

**6. Open a PR with `/open-pr`, and leave the issue In Progress.** Merges are manual here. The issue closes after the merge, verified with `gh pr view <number> --json state,mergedAt` — not on the user's word alone, and never before.

## ADR rules

**One ADR = one decision.** If a record answers two different questions, it should be two files. Mixing them means a later change of mind about one forces retiring the other along with it.

**An accepted ADR is not edited.** Changed your mind? Write a new one that supersedes it, and flip the old Status to `supersedida por 000N`. Editing in place erases the chain of reasoning, which is the entire reason the convention is worth following. Fixing a typo or a factual error is ordinary editing — that is not a change of mind.

Files are `docs/decisions/000N-titulo-em-kebab-case.md`, numbered sequentially. Check open PRs too — a number can be claimed by a branch that has not merged yet:

```bash
ls docs/decisions/ && gh pr list --state open --json number,title
```

Template:

```markdown
# 000N — Título

- **Status**: aceita
- **Data**: AAAA-MM-DD
- **Issue**: [KRA-NN](url)
- **Restringe**: issues que herdam consequências
- **Raciocínio completo**: link do documento no Linear

## Contexto
## Decisão
## Alternativa considerada e rejeitada
## Consequências
```

See `docs/decisions/0001-parametros-dataset-degrau-1.md` for a worked example, and `0002` for one that inherits a convention from it.

## What makes a record worth keeping

The template is easy; these four are what make the difference between a record that survives scrutiny and one that gets ignored.

- **Write down what was rejected, and why.** An alternative intuitive enough that someone will propose it again is an alternative that belongs in the record. Otherwise the same debate reopens with none of the reasoning.
- **Keep the honest caveat in.** If an argument supporting the decision turned out weaker than it first looked, write that down. The record exists to be questioned later, and one that hides its soft spot loses the moment someone finds it.
- **Separate principle from fitting.** Say which numbers follow from an argument and which were tuned until they looked right. Presenting everything with equal confidence leaves the next person unable to tell which one to attack when something breaks.
- **Name what is still open.** A decision with a known tension recorded against it is stronger than one that pretends to have none.

## Examples

### Example 1 — Issue labeled architecture-decision

User says: "Vamos trabalhar no KRA-24" (an issue asking for the initial dataset parameters).

1. `save_issue(id: "KRA-24", state: "In Progress")`
2. Read the issue, the domain document, `CLAUDE.md`, the repo layout
3. Analyse, then present the numbers with a recommendation and ask the genuinely open questions via AskUserQuestion
4. `save_document` → "Parâmetros do Dataset — Degrau 1" with the full reasoning; `docs/decisions/0001-parametros-dataset-degrau-1.md` with the normative values; rewrite KRA-24 as Contexto / Decisão / Consequências
5. Patch KRA-28 and KRA-29 with what they inherit
6. `/open-pr`, report the URL, leave KRA-24 In Progress

Result: three records that agree with each other, two dependent issues that know what changed, one PR awaiting manual merge.

### Example 2 — Decision with no issue of its own

User asks, mid-review: "com que frequência o gerador vai gerar novos dados?"

The answer turns out to be a decision nobody had made. Skip steps 1 and 6: write the Linear document section, write `0002-modelo-de-replay-e-geracao.md` as its own ADR — a separate question from 0001's, so a separate record — propagate to KRA-28 and KRA-29, open the PR.

### Example 3 — Superseding

User says: "o degrau 2 mudou o S, atualiza o ADR".

Do **not** edit `0001`. Write `0003-...md` stating what changed and why, set its `Status: aceita`, and change 0001's Status line to `supersedida por 0003` — that status edit is the one change an accepted ADR accepts. Update the Linear document in place, since that one evolves.

## Troubleshooting

### Linear `patch` fails with "anchor not found"

Linear escapes some characters on save — `~` is stored as `\~`. An anchor containing one will never match. Fetch the current content with `get_document` or `get_issue` and copy the anchor exactly, or pick an anchor with no special characters.

### The ADR number is already taken by an open PR

`ls docs/decisions/` only shows merged work. Check `gh pr list --state open` before choosing, and if two ADRs are in flight, say in the PR body which should merge first.

### The draft ADR says a future change will edit it

That is the convention violation this skill exists to catch. Fix it to say it will be superseded, before opening the PR.

### The decision turns out to answer two questions

Split it into two ADRs before merging. Splitting is cheap now and impossible later without retiring a record that is still half-valid.
