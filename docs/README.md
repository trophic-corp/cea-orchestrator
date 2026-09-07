# Documentation index — Trophic CEA Platform foundation

**Date:** 2026-09-07 · The repository is the source of truth for functional requirements
and technical architecture. Figma (when authorized later) is source of truth only for
visual/interaction design. ADRs live in [adr/](adr/); per-requirement pipeline artifacts
in [pipeline/](pipeline/).

## The 26 foundation deliverables → where they live

| # | Deliverable | Document |
|---|---|---|
| 1 | Existing workspace/agent audit | [product/2026-09-07-workspace-audit.md](product/2026-09-07-workspace-audit.md) §2 |
| 2 | Existing documentation audit | same, §3 (classification, decisions already made, contradictions) |
| 3 | Existing CEA knowledge audit | same, §4 (+ OCR reliability, safety-rules state) |
| 4 | Missing domain knowledge | same, §5 |
| 5 | Missing agents/skills | same, §2.2 + §6 (2 created, 4 deferred with triggers) |
| 6 | Product vision | [product/vision.md](product/vision.md) |
| 7 | User personas | product/vision.md §3 |
| 8 | Core use cases | product/vision.md §4 (UC-01…20) |
| 9 | Functional requirements | [requirements/requirements.md](requirements/requirements.md) §3 |
| 10 | Non-functional requirements | same §4 |
| 11 | CEA domain model | [domain/cea-domain-model.md](domain/cea-domain-model.md) |
| 12 | Device/control model | [iot/device-control-model.md](iot/device-control-model.md) |
| 13 | Crop/production model | [domain/production-model.md](domain/production-model.md) |
| 14 | Inventory model | [domain/inventory-model.md](domain/inventory-model.md) §2, §4–5 |
| 15 | Forecast availability model | same §1, §3 |
| 16 | R&D data model | [data/rd-data-model.md](data/rd-data-model.md) |
| 17 | Initial system architecture | [architecture/system-architecture.md](architecture/system-architecture.md) §1–5, §8–9 |
| 18 | Initial data architecture | [data/data-architecture.md](data/data-architecture.md) |
| 19 | UX information architecture | [ux/information-architecture.md](ux/information-architecture.md) |
| 20 | Security considerations | [security/security-architecture.md](security/security-architecture.md) |
| 21 | Reliability/safety considerations | architecture/system-architecture.md §6 (+ device-control-model §9; safety law in security §1) |
| 22 | Simulation strategy | architecture/system-architecture.md §7 (+ FR-SIM-*, device-control-model §10) |
| 23 | Incremental release roadmap | [roadmap/release-plan.md](roadmap/release-plan.md) |
| 24 | Open questions | [decisions/open-questions-risks-next-actions.md](decisions/open-questions-risks-next-actions.md) §1 |
| 25 | Architecture risks | same §2 |
| 26 | Recommended next actions | same §3 |

**Proposed Phase 1 scope** (awaiting owner approval): roadmap/release-plan.md, final
section.

## Architecture Decision Records (all currently `proposed` — approve/amend before R1)

| ADR | Decision |
|---|---|
| [0001](adr/0001-edge-first-local-control-autonomy.md) | Edge-first control: the room is autonomous; cloud advisory only |
| [0002](adr/0002-capability-based-device-model.md) | Capability-based device model (discovery, not hardcoded types) |
| [0003](adr/0003-recipe-immutability-and-batch-pinning.md) | Immutable recipe versions pinned by batches |
| [0004](adr/0004-mqtt-timescale-postgres-backbone.md) | RS485 fieldbus (fixed by hardware) + MQTT/Mosquitto + PostgreSQL/TimescaleDB backbone |
| [0005](adr/0005-single-tenant-now-tenant-ready-schema.md) | Single-tenant now, tenant-ready schema |

## Directory conventions

- `product/` — vision, personas, use cases, audits (point-in-time reports dated).
- `requirements/` — numbered, release-mapped FR/NFRs; the PRD-source for pipelines.
- `domain/` — entity models (CEA, production, inventory/forecast).
- `iot/` — device/control model, capability vocabulary, hardware status.
- `data/` — R&D data model + data architecture.
- `architecture/` — system architecture, reliability/safety, simulation.
- `ux/` — IA, journeys, wireframe scope (no hi-fi before review — working rule 18).
- `security/` — security architecture (normative; the `security-safety-reviewer` gate).
- `roadmap/` — release plan + Phase 1 scope.
- `decisions/` — open questions, risks, next actions (the standing separation §).
- `api/`, `firmware/`, `testing/`, `operations/` — placeholders that fill from the
  first pipelines (each has a README stating its owner and content plan).
- `adr/`, `pipeline/` — pre-existing directories (process-owned), see their READMEs.