# 0001. Edge-first control: the room is autonomous; the cloud is advisory

**Status:** proposed
**Date:** 2026-09-07
**Pipeline artifact:** docs/product/2026-09-07-workspace-audit.md (foundation analysis)

## Context

This is an operational control system for physical equipment, not a dashboard. The brief
makes reliability and local/edge safety the top concern ("the cloud/web application must
not be the only mechanism preventing unsafe physical operation"), and the hardware
engineering set already documents the answer: rack controllers "run the tier sequence
locally and hold last-known-good setpoints, so the rack keeps irrigating if the bus or
room controller drops"; the room controller owns the drain token, the 8-condition reuse
gate, dosing, HVAC/CO2 setpoints, alarms and a local time-series log with a local UI;
"the cloud owns remote view and history and nothing the room depends on"; interlocks
(leak pucks, header level switch, E-stop, MV-01 flood sensor) are hard-wired and
bus-independent. Phase A §2.5 independently argues local-first (setpoint + PID/fuzzy
locally, cloud logging layered on). The alternative architectures — cloud-first control,
or hybrid-but-cloud-authoritative — would put crop safety and flood risk behind an
internet link in a facility where connectivity is not guaranteed.

## Decision

1. **All actuation decisions execute at the edge** (rack + room controllers), per the
   allocation already documented in the engineering set. The facility platform (backend)
   and any cloud layer may propose configuration, never command hardware directly.
2. **Layered autonomy:** hardwired safety > edge control > facility platform >
   remote/cloud. Each layer must remain correct when every layer above it is absent.
   "Nothing the room depends on" is the acceptance criterion for anything placed above
   the edge.
3. Safety-envelope enforcement (bounds sourced from `safety-rules.json`) runs at the
   edge; the backend repeats validation for UX, but its rejection is advisory-fast-path
   only — the edge is the enforcing authority.
4. The room controller keeps its documented local store and local UI; the platform
   backfills telemetry gaps from it after outages (sequence-verified, gap-marked).
5. Product consequence: features are designed edge-first — a feature that cannot
   degrade gracefully to "edge keeps running, platform catches up later" is out of
   scope until that property exists.

## Consequences

- Internet loss, host loss, and backend outages are non-events for the plants (this is
  the brief's reliability bar, restated as an architectural property).
- The backend's role is exactly: configuration authoring, proposals, history, R&D,
  dashboards, alerts aggregation, sales. This keeps the cloud layer honest.
- Every release's e2e includes a "kill the cloud" test in simulation (FR-SIM-02).
- Costs: edge must persist desired state + schedules (non-volatile), run validation
  logic, and buffer telemetry — firmware complexity is accepted and budgeted; console UX
  must represent edge-authority honestly (stale badges, "edge-autonomous" banners).

## Alternatives considered

- **Cloud-first (thin edge):** simplest platform, but a fiber cut becomes a crop event;
  contradicts the documented hardware design and the brief's reliability requirement.
  Rejected.
- **Hybrid, cloud-authoritative (edge executes cloud-computed setpoints):** still
  couples actuation correctness to connectivity; the documented architecture already
  rejects this ("nothing the room depends on"). Rejected.
- **Edge-only (no cloud/platform):** matches reliability but abandons the brief's
  actual product requirements (portal, R&D, forecasting, multi-user). Rejected.
- **Hardwired-only PLC control with software as pure observer:** the most conservative,
  but abandons recipe/batch/R&D value and future automation; also not what the
  engineering set specifies. Rejected — but hardwired safety remains a *layer within*
  the chosen architecture (the one part of this alternative we keep, unchanged).