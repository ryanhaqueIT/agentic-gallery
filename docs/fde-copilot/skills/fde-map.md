# /fde-map — inside-out discovery

> **Status:** specified, not yet built. Design spec for the runnable skill; the dashboard's
> systems map, coverage grid, and signals ledger are this stage's outputs, prototyped.
> Refine via PR — this file is the source of truth.

## Purpose

Turn whatever the customer hands over into the **Customer Context Pack**: an evidence-cited
map of their systems, auth, integration points, pains, and numbers — plus the regenerated
engagement dashboard. Deterministic structure carries the facts; the model carries the
judgment; a human gate carries the risk.

## Invocation

```
/fde-map <customer-slug>
```

Re-runnable: fire it again whenever new inputs land in `engagements/<customer>/inputs/`.

## Reads / Writes

| | |
|---|---|
| **Reads** | `01-brief.md` + `inputs/` (repo pointers, API docs, data cuts, call notes, transcripts) |
| **Writes** | `02-context-pack/` — `pack.md`, `inventory.json`, `domain-model.md`, `baseline.md`, `conversation-context.md` — + regenerates the dashboard (systems map, coverage grid, signals) |

## What runs inside

1. **Branch on what exists** (graceful degradation — never guess):
   - repo access → deterministic scans: tree · dependency manifests · API routes · env vars · webhooks · auth
   - API docs → endpoint + auth inventory
   - data cut / notes → schema + workflow extraction (their field names = their ontology)
   - **transcripts & conversations → the TRANSCRIPT GATE** (below)
   - nothing → gap list → new interview questions
2. **Transcript gate** — the most sensitive input, so it's gated:
   consent + PII scrub confirmed → extract personas, intents, objections, edge cases,
   vocabulary, actual-vs-documented workflow → **you review** the extracted summary
   before it enters the pack as `conversation-context.md`.
3. **One-writer synthesis** — parallel readers, single merge (parallel writers make
   conflicting implicit decisions). Every claim carries file:line, doc page, or
   run/timestamp citation.
4. **Entity-verify pass** — every named system, route, and field checked against its
   source. Kills hallucinated architecture before it ships.
5. **Coverage + signals update** — extracted insights land in the signals ledger
   (quote-anchored, deduped, lifecycle: new → corroborated → validated) and file into
   the MEDDPICC + T coverage grid. Suggested and validated never look the same.

## Gate

You review the pack — and the empty/partial coverage cells generate the ranked
**ask-next queue** for your next conversation.

## AIDO state

**DONE** — you're looking at its output; every map claim cites a source.
