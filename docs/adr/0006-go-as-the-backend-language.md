# 0006. Go as the backend implementation language

**Status:** proposed
**Date:** 2026-09-08
**Pipeline artifact:** docs/pipeline/oq16-backend-language-spike/
**Resolves:** [OQ-16](../decisions/open-questions-risks-next-actions.md)

> **Why `proposed` and not `accepted`.** The decision itself is settled — see below. It is
> held at `proposed` only until [OQ-3](../pipeline/oq3-room-controller-platform/00-decision-brief.md)
> closes, so the ADR can record the room-controller platform as context rather than as a
> pending question. **OQ-3 cannot reverse this** (§Consequences), so backend work in Go is
> not gated on it.

## Context

The Phase 1 backend (registry, ingest, alerts, backup, read API) needed a language. Node.js
was never chosen — it was **inherited from the bootstrap scaffold's CI/compose detection**
and had never been weighed. The owner flagged a preference for Go.

The evaluation recorded in OQ-16 (2026-09-07) established what actually differentiates the
two, and — importantly — what does not:

- **Throughput is not a factor.** Design headroom is single-digit messages/second. Both
  languages clear that by orders of magnitude. Any benchmark either would "win" is noise.
- Genuine differentiators: runtime/ops fit on an unattended facility host (Go), API and
  JSON-Schema tooling breadth plus full-stack TypeScript (Node), possible Linux-SBC
  room-controller synergy (Go, contingent on OQ-3), hiring pool (Node), and **owner
  fluency — named decisive**.

The architecture is language-agnostic by construction: MQTT/TLS (ADR-0004),
PostgreSQL/TimescaleDB (ADR-0004), the capability model (ADR-0002), org-scoping
(ADR-0005), and OpenAPI boundaries are all fixed independently of language. The choice
affects the CI job and the compose build context, and nothing else architectural.

A two-arm spike was planned to decide this ([the spike protocol](../pipeline/oq16-backend-language-spike/00-spike-protocol.md),
scorecard fixed 2026-09-08). That protocol stated that a clear owner-fluency answer **wins
outright** over criteria 2–5, on the explicit grounds that this is a small team with a bus
factor of 1 (AR-13).

## Decision

**The backend is written in Go.**

**The two-arm spike was not run.** At Phase 1 kickoff (2026-09-08) the owner gave the
decisive criterion directly: they are fluent in Go. Building a Node arm to inform a
criterion that was already settled would have produced evidence for a question no longer
being asked. The spike protocol anticipated this outcome and pre-authorised it; collapsing
it is an application of the protocol, not a departure from it.

This is recorded plainly rather than dressed up as a spike result. **No benchmark was run,
no Node arm exists, and no measured comparison backs this ADR.** The justification is
owner fluency on a bus-factor-1 team, which OQ-16 named decisive before any code was
written.

## Consequences

**Easier**
- Single static binary on the facility host: no runtime, no `node_modules`, trivial
  container, fast cold start after a power event (R12/R13 of the OQ-3 brief).
- The owner can debug production at 2 a.m. during a grow — the criterion that decided it.
- Compile-time typing catches a class of agent-authored regressions before runtime, which
  matters disproportionately here because most code will be agent-authored.
- If OQ-3 lands on a Linux SBC or IPC, **one language spans room controller and backend**.

**Harder**
- Frontend is TypeScript, so the stack is not single-language. Shared types must be
  generated from the OpenAPI spec rather than imported — **the OpenAPI schema becomes the
  contract of record**, which is arguably the more honest arrangement.
- Smaller hiring pool locally than Node. Accepted; the bus-factor-1 reality that decided
  this also makes the hiring-pool argument largely theoretical today.

**Must now be respected rather than re-derived**
- Phase 1 backend build proceeds in Go. `docker-compose.prod.yml` and
  `.github/workflows/build-test-deploy.yml` need their Node assumptions replaced
  (`build-deploy-engineer`).
- `docs/architecture/system-architecture.md` §2 says *"backend (Node.js, single modular
  service)"* — superseded by this ADR; the module boundaries it lists are unaffected.
- `backend-engineer`'s scaffold assumptions change; the module boundaries do not.

**Cannot be reversed by OQ-3.** Linux SBC or IPC reinforces Go; facility-host collapse is
neutral; a split MCU+SBC design is neutral. **No OQ-3 outcome favours Node** — which is why
this ADR is drafted ahead of that decision.

## Alternatives considered

**Node.js/TypeScript** — the incumbent by accident. Real advantages: broader JSON-Schema
and OpenAPI tooling, full-stack TypeScript with the console, larger hiring pool. Rejected
because the decisive criterion (owner fluency) points the other way, and because its
strongest argument — shared types with the frontend — is answerable with OpenAPI codegen.

**Run the full two-arm spike anyway, for rigour.** Rejected as ceremony. The spike existed
to produce an owner-fluency judgement; that judgement arrived without it. Building a Node
arm to be discarded would have cost days of Phase 1 week 1 and changed no outcome. The
cost of being wrong is bounded — the architecture is language-agnostic, so reversing this
is a rewrite of one service, not a redesign.

**Defer until OQ-3 closes.** Rejected: OQ-3 can only reinforce or be neutral toward Go, so
waiting would stall the backend build for information that cannot change the answer.

**Rust / Python.** Not seriously considered. Rust: no owner fluency, and its advantages are
irrelevant at this scale. Python: the deployment-footprint argument that favours Go over
Node applies to it with more force.
