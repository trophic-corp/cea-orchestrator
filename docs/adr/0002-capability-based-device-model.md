# 0002. Model devices by advertised capabilities, not hardcoded device types

**Status:** proposed
**Date:** 2026-09-07
**Pipeline artifact:** docs/product/2026-09-07-workspace-audit.md (foundation analysis)

## Context

The platform must control a heterogeneous and *evolving* fleet: today's Rack A hardware
(per-tier LED bars, flood-and-drain solenoids, sensors — exact inventory per Phase D BOM
and Rack A Specification), tomorrow's revised racks, optional add-ons (CO2, HVAC,
metering), and eventually a different product family entirely (aquarium/aquascaping
equipment: programmable tank lighting, sensor/controllers). Working rule: the software
architecture should accommodate **hardware capability discovery rather than hardcoding
every device**, and the CEA and aquarium products must be able to share the device
platform while keeping separate UX. Phase A/B research also makes clear that not every
parameter is actuated or even sensed today — the UI must not promise controls that
hardware can't deliver, and development must proceed against virtual hardware before
physical hardware exists (FR-SIM-01).

## Decision

1. Every device carries a versioned **capability profile** — a JSON descriptor
   (schema-versioned, validated at provisioning) advertising capability classes it can
   perform, with per-class parameters (e.g., `light.dim {channels, min, max}`).
   Capability classes are a small, registry-defined vocabulary shared across products
   (lighting, irrigation, sensing, climate…); classes are product-agnostic.
2. The platform's control, UI, and recipe-application logic is written **against
   capability classes, never device SKUs**. A zone's capabilities are the union of its
   devices' advertised profiles; a control appears only when the capability exists
   (FR-CTL-06); a recipe parameter is applied or explicitly recorded as "not applied —
   lacks capability" (production-model §4.5).
3. Profiles are **forward-compatible**: an unknown capability class or newer profile
   schema version is recorded and ignored (never a crash, never a silent config), and
   upgraded when the platform learns the class.
4. Device *types* remain only as descriptive metadata + commissioning guidance, not as
   behavior switches.
5. The same capability vocabulary serves the future aquarium product: a programmable
   tank light is a `light.dim` + `light.schedule` device; its product surface is
   separate (vision §2), its platform integration is none new.

## Consequences

- New device types (including another company's or product line's) integrate without
  core-code changes — only a profile schema registration and commissioning flow.
- The recipe model, command pipeline, and UI need one generality path, built once.
- Virtual devices (simulation, FR-SIM-01) are just devices whose profile executes
  against a simulator — no forked control logic.
- Cost: every command path must handle "capability absent/incompatible" as a first-class
  outcome, and profile schema versioning must be maintained with real discipline.
- Fleet queries ("which zones can't sense CO2?") need the extracted capability index
  (data-architecture §4), not profile parsing at query time.

## Alternatives considered

- **Hardcoded device-type tables** (controller model X → known IO map): simplest to
  build; breaks on the first hardware revision, forces schema migrations for new
  devices, and forks the aquarium product from day one. Rejected.
- **Full plug-in device drivers (OO style)**: overkill at this fleet size; the capability
  vocabulary gives the same extensibility with less machinery. Deferred unless the
  vocabulary proves too coarse.
- **Capabilities negotiated via runtime introspection only (no stored profile)**:
  loses the ability to plan/validate zone capability before device boot, complicates
  simulation, and loses fleet-wide queries. Rejected.