# Incremental release roadmap

**Status:** **Phase 1 (Release 1) — accepted and locked 2026-09-08** (owner go-ahead,
including the week-1 spike gate). Releases 2–8 remain **proposed** — shape agreed,
scope not yet locked; each is scoped for real at its own kickoff.
**Date:** 2026-09-07 · **Phase 1 locked:** 2026-09-08
**Deliverable mapping:** incremental release roadmap of the foundation analysis. Release
boundaries follow dependencies and risk, not the brief's example numbering — recipes
move to R3 (they exist to be *pinned by batches*; earlier placement creates a throwaway
model), and R4 combines production-ops + inventory + forecast because forecast
confidence is meaningless without harvest actuals feeding it.

Hardware tracks in parallel (owned by the hardware program, not this platform): rack
manufacturing (Rack A MFG Rev 2), room fit-out (11 racks), HVAC/dehumidifier install,
Phase-2 plenum retrofit. Platform releases are shaped so software is never blocked on
that hardware (simulation contract), but each release's *production* validation names
the hardware it needs. Owner decisions 2026-09-07: the engineering set is **draft-stage
(initial idea)** — revisions are expected and are absorbed by the capability model, never
hardcoded; crop rollout order is **microgreens first, aquatic production after the first
microgreen rollout** (aquatic zone engineering + aquatic protocols are out of R1–R4
scope).

---

## Release 0 — Foundation (this deliverable)
**Objective:** requirements, domain model, architecture decisions on record; approved.
**Users:** program stakeholders. **Content:** this docs/ set + ADR-0001…0005 + audit.
**Acceptance:** owner reviews; open questions answered or explicitly deferred; ADR
statuses move to accepted as merged. **Not included:** any code.

## Release 1 — Facility, devices, telemetry (proposed Phase 1 — detail below)

## Release 2 — Control & schedules
| Field | Definition |
|---|---|
| Objective | Operator can configure zones and the edge executes reliably without cloud; desired-vs-actual visible end-to-end. |
| Users | P1 operator, P4 technician. |
| Features | FR-CTL-01..07 (desired-state store, edge schedule execution, command/ack/retry pipeline, time-boxed overrides, safety-envelope enforcement at edge, capability-aware UI, emergency-state reflection), FR-ALR-05 (device failure alerts), FR-DEV-05/06 (command history, firmware inventory). |
| Dependencies | R1 registry + ingest; rack controller firmware baseline (below); at least one physical rack controller OR virtual facility (both, ideally). |
| Backend | Config authoring + proposal publishing via broker; ack/state ingestion; override & audit. |
| Frontend | Tier detail control panel (capability-aware), override banner, command lifecycle view. |
| Firmware | Rack controller: schedule executor (photoperiod, flood sequence), fan PWM, desired-state persistence, command/ack over bus; room controller: bus master, MQTT bridge, validation authority. |
| Hardware | ≥1 rack (solenoids, fan, LED dimming pair + any available LED bar); room controller board. Photoperiod mechanism depends on LED driver choice → [OQ-14]. |
| Data | Desired-state history from day one (R&D irreducible). |
| Testing | "Kill the cloud" scenario in sim; command retry/timeout matrix; override auto-revert; e2e per PRD. |
| Acceptance | A tier runs its photoperiod+flood schedule for 72 h with the backend stopped; commands show full ack lifecycle; override expires and reverts with audit; out-of-envelope command rejected at edge with reason. |
| NOT included | Recipes (R3), any closed-loop regulation (R7), OTA rollout UX beyond inventory view. |

## Release 3 — Recipes & production batches
| Field | Definition |
|---|---|
| Objective | Versioned recipes pin to batches; full batch lifecycle with provenance; harvest facts captured. |
| Users | P2 grow manager primary, P1 executing. |
| Features | FR-RCP-01..06, FR-BAT-01..06. |
| Dependencies | R2 (desired state to apply recipes to); ADR-0003 enforcement. |
| Backend | Recipe/version services (immutability, hashing, lineage), batch lifecycle, zone assignment, event log, harvest records. |
| Frontend | Recipe library/editor (capability-aware, envelope-validated), batch board + detail timeline, harvest flow (mobile-first), production calendar. |
| Firmware | None new (recipe application = desired-state config push). |
| Hardware | None new (runs on R2 fleet). |
| Data | The R&D irreducibles complete: pinned versions + assignments + desired-state history + harvest actuals. |
| Testing | Immutability policy tests (tamper detection), hash verification, batch↔telemetry join integrity; e2e harvest→lot handoff (with R4 stub). |
| Acceptance | Operator plans→runs→harvests a real batch end-to-end; historical batch answers "which exact recipe version, which zones, what environment, what yield" from the system alone; recipe edits never affect past batches. |
| NOT included | Inventory economics (R4), forecasts (R4), aquatic crop support entirely — deferred per owner sequencing 2026-09-07 (aquatic production starts after the first microgreen rollout; lifecycle question moves to that phase). |

## Release 4 — Production ops, inventory, forecast availability
| Field | Definition |
|---|---|
| Objective | Harvests become lots; resources measured/estimated; availability = stock + risk-adjusted forecast − commitments. |
| Users | P1, P2, P3. |
| Features | FR-INV-01..04, FR-FCT-01..03, FR-TEL-06 (metering where hardware allows), FR-ANL precursors (per-batch resource views). |
| Dependencies | R3 (batches/harvests); water-recovery telemetry ingestion (skid node). |
| Backend | Lot management, reservations, forecast revision chain, confidence policy (v1 parameters — domain-reviewed), resource estimation with provenance. |
| Frontend | Inventory view, availability timeline, forecast-vs-actual, order-impact preview (internal). |
| Firmware | Water-skid/terrace node telemetry if not already in R1 scope. |
| Hardware | Water recovery skid operational (for real consumption data); per-rack metering absent → estimates marked as such. |
| Data | Actual-vs-expected deltas start accumulating (confidence feedback loop). |
| Testing | Availability arithmetic property tests (never > physical+forecast; netting atomicity); abort-impact workflow; confidence floor enforcement. |
| Acceptance | Operator can answer "what can I promise on date D" with confidence floors honored; an aborted batch triggers order-impact alerting in an internal dry-run. |
| NOT included | Customer-facing portal (R5), pricing/invoicing, ML anything. |

## Release 5 — B2B portal
| Field | Definition |
|---|---|
| Objective | Customers browse real+forecast availability and order, without any way to oversell physics. |
| Users | P5 customers; P3 monitors. |
| Features | FR-ORD-01..05; portal identity domain (security-architecture §2). |
| Dependencies | R4; remote-access decision for serving the portal off-LAN → [OQ-12]; VPN/relay or cloud-publish decision. |
| Backend | Portal API + accounts + order/reservation flows (reusing R4 core). |
| Frontend | Separate portal app (browse, availability-by-date, custom packs, order status). |
| Firmware | None. |
| Hardware | None (but portal serving needs reliable connectivity decision). |
| Testing | Oversell impossibility under concurrent orders; cancellation-window rules; shortfall exception flows. |
| Acceptance | A customer orders against a future harvest with confidence floor respected; cancelled/aborted batch surfaces to orders as exceptions; operator console shows commitments. Zero portal→console feature leakage. |
| NOT included | Payments/invoicing integrations ([OQ-13]), delivery/logistics, cold-chain, recurring auto-orders. |

## Release 6 — R&D analytics
| Field | Definition |
|---|---|
| Objective | The brief's R&D questions answerable without writing code; exports for external analysis. |
| Users | P2, P3. |
| Features | FR-ANL-01..03 (+FR-TEL-07 if imaging hardware ever lands — currently ABSENT). |
| Dependencies | ≥2–3 real batches harvested per recipe lineage (data reality, not blockers). |
| Backend/Frontend | Query layer (recipe/rack/variety comparisons, resource-per-kg with provenance display), saved reports, CSV/JSON export. |
| Testing | Result correctness vs hand-computed joins on golden datasets; provenance always rendered. |
| Acceptance | "Which recipe version yielded most for crop X?" and "water+electricity per kg for batch Y?" answered in <30 s from the console. |
| NOT included | Predictive modeling (R8), imaging analytics, cross-facility benchmarking. |

## Release 7 — Advanced automation
| Field | Definition |
|---|---|
| Objective | Closed-loop regulation where capability + sourced targets exist — one loop at a time, domain-reviewed. |
| Users | P1/P2. |
| Features | FR-CTL-08/09: room HVAC & dehumidifier integration, CO2 PID enrichment (already hardware-supported, currently room-controller-local), VPD-derived control, energy-aware schedule optimization. |
| Dependencies | HVAC/dehumidifier installed; **EC operating target TBD resolved** (`safety-rules.json`) before any dosing-loop tightening; loop-by-loop domain review (botany + electronics). |
| Backend/Frontend | Loop policy versioning, per-loop enable/disable, loop health metrics (oscillation, duty). |
| Firmware | Room controller loop implementations per reviewed policies. |
| Testing | HIL + sim fault matrix per loop; loop-kill switch; envelope regression. |
| Acceptance | Each enabled loop holds its sourced target band under sim perturbation with interlocks intact; disabling a loop degrades to schedule-only safely. |
| NOT included | Anything per-tier climate (hardware ABSENT by design), ML-based control. |

## Release 8 — Predictive analytics / optimization
| Objective | Expected-yield models and schedule optimization, **only after** R6 demonstrates data quality and enough batches exist. | 
|---|---|
| Dependencies | Data-quality review (provenance hygiene, calibration discipline); owner go-decision. |
| Features | FR-ANL-04; forecast confidence upgrade from statistics to learned models (still never "guaranteed"). |
| NOT included | Autonomous control via ML (out of scope for this roadmap entirely). |

---

## Phase 1 (Release 1) — foundation, small and validatable — **LOCKED 2026-09-08**

> **Scope lock (owner, 2026-09-08).** The scope below is fixed as written, including the
> week-1 spike gate. Additions go to a later release, not into Phase 1. Two notes carried
> in from the handover:
> - **Saffron is out** ([OQ-1] closed) — no Phase 1 impact, since Phase 1 contains no
>   crop-specific logic at all. Recipes remain R3 and are untouched by that decision.
> - **[OQ-16] is closed: the backend is Go** ([ADR-0006](../adr/0006-go-as-the-backend-language.md)).
>   The two-arm spike was collapsed on the decisive owner-fluency criterion rather than
>   run. Deliverable 1's "week-1 language spike gate" below is therefore **satisfied, not
>   skipped** — but note it was satisfied by a judgement, not by evidence.
> - **[OQ-3] (room controller compute platform) is still open**, and the owner has
>   **paused the Phase 1 build behind it** (2026-09-08) rather than building on an
>   assumption. [Decision brief](../pipeline/oq3-room-controller-platform/00-decision-brief.md)
>   is ready; it closes as ADR-0007. This is the critical path.

**Objective:** the platform's spine exists end-to-end and is proven on one rack + the
virtual facility: registry, device/capability model, telemetry ingest/store/serve, basic
operator visibility, alerting v1, backup v1, simulation v0 — with org-scoping and audit
from the first commit.

**In scope (FR IDs):** FR-FAC-01..05, FR-DEV-01..04, FR-TEL-01..05, FR-ALR-01..04+06,
FR-SIM-01 (v0: virtual rack+room controller), FR-USR-01..04, FR-BKP-01..02 (+03 basic
scheduled), NFR-REL-01/02 architecture shape (edge path exists, no cloud dependency
created), NFR-PER-01/02, tenant-scoping tests (ADR-0005).

**Concrete deliverables:**
1. Backend service modules: authz, registry, ingest (MQTT client), alerts, backup —
   one service, Postgres+Timescale, Mosquitto wired in `docker-compose.prod.yml`.
   **Week-1 gate: language spike [OQ-16] — CLOSED 2026-09-08 → Go
   ([ADR-0006](../adr/0006-go-as-the-backend-language.md)).** The planned two-arm build
   (Go and Node.js/TS against the sim harness) was collapsed when the owner supplied the
   decisive criterion — fluency — at kickoff. The service is Go; compose and the CI job
   need their inherited Node assumptions replaced.
2. Operator console v0: login, Home (facility status), Grow drill-down (rack/tier
   snapshots), Devices (registry/health/commissioning), Alerts (center), Settings
   (users, backup/export).
3. Firmware v0: rack controller skeleton (capability advert, telemetry publish, command
   ack) + **virtual device profiles** so the full stack runs in CI with zero hardware.
4. Sim harness v0: 1 virtual rack (4 tiers) + fault switches (offline, stale, implausible).
5. Deploy: compose stack runs on the facility host (or dev box); GH workflow deploy step
   wired by `build-deploy-engineer`; backup job + restore drill documented.

**Validation (acceptance criteria):**
- Virtual 11-rack facility (and ≥1 real rack controller if hardware ready) streams
  telemetry for 72 h with zero gaps unaccounted (gaps must be recorded gaps).
- Commissioning flow claims a device, discovers capabilities, assigns a tier, and a
  test command round-trips with ack visible.
- Home answers "anything wrong?" in <5 s; every number shows source age.
- Org-scoping test suite passes (every repository method).
- Export bundle from a populated system restores into a clean system with verification.
- Security review (`security-safety-reviewer`) passes on the authz/audit/transport layer.

**Explicitly NOT in Phase 1:** any actuation (no command to a real valve/LED/light —
telemetry and registry only), recipes, batches, inventory, portal, closed loops, OTA.

**Why this is the right first slice:** it exercises every architectural seam (identity,
capability model, transport, storage, UI, simulation, backup, deploy) with near-zero
physical risk — nothing in Phase 1 can actuate hardware — while producing the operator
glance-visibility that justifies the platform's existence to its first user.

**Effort shape:** backend ~registry+ingest+alerts; frontend ~5 screens; firmware ~1
skeleton + sim profiles; infra ~compose+deploy+backup. Each subsequent release reuses
every one of these seams unchanged.