# /fde-demo — build the demo on the platform

> **Status:** specified, not yet built. Design spec for the runnable skill; the demo-spec
> view is this stage's output artifact, prototyped. [Refine it on GitHub](https://github.com/ryanhaqueIT/agentic-gallery/blob/master/docs/fde-copilot/skills/fde-demo.md).

## Purpose

Demo-as-spec, on the customer's own data, within days: turn the context pack into an
**approved demo spec**, then build the **actual live agent** on the HappyRobot platform
through its MCP tools — plan, validate, then and only then execute.

## Invocation

```
/fde-demo <customer-slug>
```

## Reads / Writes

| | |
|---|---|
| **Reads** | `02-context-pack/` (inventory, domain model, baseline, conversation context) |
| **Writes** | `03-demo.md` + demo-spec view + a **live demo agent** on the platform (dev environment) |

## What runs inside

1. **Draft the demo spec** — use case, persona, tools, data plan, minute-by-minute script
   ("do the last thing first": the proof moment lands in the first 90 seconds) —
   then **STOP for your approval**. Nothing is created on the platform before sign-off.
2. **On approval, build via MCP** — template workflow → persona prompt → Twin table loaded
   with the customer's CSV → tool nodes (mock endpoint first, real later).
3. **Start knowledge-base chunking early** (10–15 min lead time) while wiring the rest.
4. **Verify** — `test_node` / `test_all` waves, auto-generated eval northstars, then a
   **$0 web-call link** (browser WebRTC — no phone number needed for the demo).
5. **Honest framing built in** — the spec carries a proves / doesn't-prove-yet split and
   mock flags, so a wired-but-mocked tool is never presented as live.

## Gate

Two: your approval of the spec **before** any platform write, and the verification
checklist **before** any human sees the demo.

## AIDO state

**IN PROGRESS** — Emma v5 live in Production with a Completed run on record, but
**gated**: run `#9c6cf636` greeted generically. Verify persona hydration (does `to_number`
flow on web calls? does the prompt consume `{{persona.response.*}}`?) before demoing.
