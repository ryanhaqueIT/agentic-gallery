# /fde-integrate — the production path

> **Status:** specified, not yet built. Design spec for the runnable skill; the AIDO
> integration is the living proof of the pattern. Refine via PR.

## Purpose

From inventory flags to a **reversible go-live**: build the doorways, chase the
credentials early, ship everything dark, and make going live a single, instantly
undoable switch. The rollback story is part of the sale.

## Invocation

```
/fde-integrate <customer-slug>
```

## Reads / Writes

| | |
|---|---|
| **Reads** | `02-context-pack/` (inventory flags) + `03-demo.md` (integration seams the demo exposed) |
| **Writes** | `04-integration/` — adapter scaffold, credential checklist, go-live runbook, cutover plan |

## What runs inside

1. **Adapter scaffold (Option B pattern)** — the platform speaks plain HTTPS only (no
   SigV4, no SDKs), so blocked systems get a thin doorway: API Gateway + small Lambda,
   API-key in, IAM invoke out. TDD'd, isolated stack, reuses 100% of existing logic.
2. **Credential checklist** — every ⚠ lead-time flag from the inventory becomes a row
   with an owner and a clock, chased in week one (they kill timelines in week six).
3. **Ship dark** — everything deploys inert: flags default OFF, adapters 503/undeployed,
   webhook URLs empty. Deployment and go-live become two separate events; the risky one
   happens while nothing depends on it.
4. **Go-live runbook** — what flips, in what order, how it's verified, how it rolls back,
   who's on point.
5. **Flip ONE switch** — for voice: the number's routing target (re-aim, not re-buy).
   Rollback = flip it back. Instant, no deploy, no data loss.

## Gate

You (or the customer's ops) review the runbook. Nothing flips without the sign-off meeting.

## AIDO state

**PARTIAL** — persona adapter LIVE ✓ (7/7 tests, proven end-to-end); calendar/post-call
shims + SIP path pending; every prod-touching flag still OFF; the Twilio voice URL still
points at GCP — which is exactly the point.
