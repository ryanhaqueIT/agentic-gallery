# /fde-prep — outside-in discovery

> **Status:** specified, not yet built. This is the skill's design spec, transcribed from the
> approved fde-toolkit blueprint. The runnable skill (`.claude/skills/fde-prep/SKILL.md`)
> gets built from this document. Found a gap? Open a PR — this file is the source of truth.

## Purpose

Everything you can do **before the customer gives you anything**: research the account,
form the use-case and ROI hypothesis, and — the Palantir lesson — send the **data-access
ask on day one**, because data-access politics (not engineering) is what kills pilots.

## Invocation

```
/fde-prep <customer-slug>
```

Human-invoked only (`disable-model-invocation: true`). The agent never fires a stage on its own.

## Reads / Writes

| | |
|---|---|
| **Reads** | nothing but a customer name |
| **Writes** | `engagements/<customer>/01-brief.md` + **dashboard v0** (the engagement's visible surface from day one — mostly empty, showing the flow and the unknowns) |

## What runs inside

1. **Parallel research subagents** — company, vertical, tech-stack signals (TMS in job ads,
   engineering blog), stakeholders. Each returns a distilled, cited summary.
2. **Use-case + ROI hypothesis** — capture · labor · margin, sized with public numbers,
   clearly marked as hypothesis until the customer confirms.
3. **Interview kit** — baseline questions (the ROI denominator set) + system-access
   questions derived from the platform's integration inventory, Mom-Test style
   (past-specific, quantifying, commitment-seeking).
4. **Day-one data-access ask** — the email requesting a static data cut, API docs,
   call recordings/chat exports, and flagging the lead-time landmines
   (admin-installed apps, connected-app approvals, support-gated dependencies).

## Gate

You review `01-brief.md` before anything is sent. The ask ships day one — but only after
your read.

## AIDO state

**DONE** — context established from the engagement notes; dashboard v0 superseded by the
live dashboard you're reading this from.
