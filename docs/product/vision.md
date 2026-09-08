# Product vision — Trophic CEA Platform

**Status:** proposed (pending owner approval) · **Date:** 2026-09-07
**Deliverable mapping:** product vision (§2), user personas (§3), core use cases (§4) of the
foundation analysis. Owner: Principal Product Architect. Review: `product-strategy-specialist`,
`ux-product-designer`.

## 1. Context

Two companies share one platform:

- **Company A — Sholaverde (Ooty/Nilgiris)** operates the CEA facility: premium microgreens
  (~70–80% of production), aquatic plants (~20–30%), exotic plants (optional, ~10%). It
  needs production, crop/batch management, inventory, harvest/yield tracking, R&D
  analytics, B2B sales and order management, production forecasting, and a customer
  availability portal.
- **Company B — Trophic (manufacturing base Coimbatore)** manufactures the infrastructure
  Sholaverde uses and intends to sell to external customers: CEA racks, LED lighting,
  sensors, controllers, pumps, flood-and-drain irrigation, ventilation equipment, plus an
  aquarium/aquascaping equipment line with its own future software suite.

The existing research (`knowledge/cea/`) was written for a three-crop program
(microgreens + aquascaping plants + saffron) with co-equal microgreen/aquatic zones. The
current direction is microgreens-primary. This document treats the current direction as
authoritative and the research as evidence.

**Resolved 2026-09-08 ([OQ-1](../decisions/open-questions-risks-next-actions.md), owner):
saffron is deferred indefinitely.** The program is microgreens first, aquatic production
after the first microgreen rollout, exotic optional. Saffron material in `knowledge/cea/`
and `safety-rules.json` is retained as reference, not active scope — re-entry is an owner
decision plus a recipe definition, not a redesign (the model is crop-agnostic per
ADR-0002/0003).

## 2. Product vision

> **The Trophic CEA Platform is the operating system for a rack-based controlled-environment
> grow facility: it lets one operator run many independently configured growing zones
> reliably, keeps every batch's full history explainable, and turns that history into
> production forecasts a B2B customer can order against — with the safety of the physical
> plant never depending on an internet connection.**

Four commitments, in priority order:

1. **Operational reliability first.** This is a control system with a dashboard attached,
   not a dashboard with devices attached. Local control loops, schedules, interlocks and
   documented fail-safe states run at the edge; the cloud layer advises, aggregates, and
   sells. (Grounded in Phase A §2.5 "local-first" and Rack A Specification §07.)
2. **One operator, many zones.** A single person must be able to glance at the facility
   and know: is anything wrong, what needs doing today, and is anything at risk in the
   next 48 hours. Racks, tiers and crops differ; the model must not assume uniformity.
3. **Every gram is explainable.** Historical production data must always identify the
   exact recipe version, environment history, rack/tier, inputs and outcomes for any
   batch — the foundation the R&D roadmap (and, much later, ML) builds on.
4. **Forecast ≠ inventory.** Future harvests are sellable capacity, subject to confidence
   and reservation rules — never guaranteed stock. The B2B portal exposes risk-adjusted
   availability, not raw expected yield.

**Platform/product separation.** The CEA platform and the future Aquarium/Aquascaping
platform are **separate products** with separate UX. They may share platform
capabilities: device identity/provisioning, telemetry pipeline, command pipeline,
configuration/recipe versioning, alerting, observability, audit. Architecture decision:
[ADR-0002](../adr/0002-capability-based-device-model.md) — the device layer is
capability-based precisely so an aquarium light or pump controller is just another device
profile. Shared *infrastructure*, never shared *UX*.

**Commercial position** (from Phase B/C evidence): no Indian vendor currently sells a
sealed-room, multi-zone CEA monitoring/control platform (Fasal/CropIn are open-field).
Product 1.6 (monitoring node + dashboard) is the catalog's software anchor with recurring
margin; the platform makes Trophic hardware more valuable and each hardware sale makes
the platform more entrenched. Productization to external farms is a *constraint to
design for*, not a feature to build now.

**What this platform is NOT:** not a general farm-management app, not an open-field IoT
kit, not an aquarium controller (that's the future aquarium suite), and not an AI
grower. No ML until the data model is proven (working rule 17).

## 3. User personas

| # | Persona | Who | Primary needs | Product surface |
|---|---------|-----|---------------|-----------------|
| P1 | **Operator/Maintainer** (primary) | The person physically running the Ooty room daily — seeding, monitoring, harvesting, cleaning, responding to alerts. Likely 1–2 people total. | Facility health at a glance; today's tasks; alert triage in <10s; flows usable on a phone with wet hands and marginal connectivity; desired-vs-actual always visible; manual override with confirmation | Operator console (mobile-tolerant web) |
| P2 | **Grow manager / R&D lead** | Designs crop plans and recipes, tunes setpoints, reviews experiments. Initially the same person as P1 — must not be blocked by role separation, but their *tasks* differ. | Recipe editor with versioning; batch planning & assignment; per-batch environment review; R&D comparisons (recipe A vs B, rack A vs B) | Operator console (desktop-oriented views) |
| P3 | **Business owner (Sholaverde leadership)** | Decides what to grow, what to sell, at what price. | Production status vs plan; yield, waste, water/energy per kg; order book; forecast confidence | Operator console (read-mostly) + exports |
| P4 | **Device technician (Trophic)** | Commissions and maintains hardware — racks, controllers, sensors, firmware. May be remote (Coimbatore) while hardware is in Ooty. | Commissioning workflow; device health/fleet view; firmware/OTA with rollback; diagnostics (why did this device drop?); physical layout awareness | Operator console "Devices" area + tools |
| P5 | **B2B customer** | Chefs, restaurants, hotels, retailers buying microgreens on recurring or ad-hoc schedules. | Browse what's available *now and forecast*; order with delivery dates; custom packs; order status | **Separate** B2B portal (no operator features) |
| P6 | **External farm operator** (future, after productization) | An operator at another facility running Trophic hardware. Same needs as P1, different tenant. | Same as P1, isolated tenant data | Same console, tenant-scoped |
| P7 | **Aquarium/aquascaping customer** (future) | Aquarist using Trophic aquarium hardware. | Tank control, schedules, telemetry for aquarium equipment | **Separate product** (future aquarium suite) |

Explicitly *not* personas yet: agronomy consultant (P2 covers it at this scale), multi-site
fleet manager (one facility now), ML engineer (no ML yet).

## 4. Core use cases

Ordered by priority; each maps to functional requirements (see
[requirements.md](../requirements/requirements.md)) and to a release in the
[roadmap](../roadmap/release-plan.md). The actor is P1 unless noted.

**Operate & monitor**
- **UC-01 — Assess facility health at a glance:** "Is anything wrong, and what needs me
  today?" One screen: active alarms, off-target zones, batches due (harvest/next step),
  device issues, connectivity status. (FR-ALR-*, FR-FAC-*)
- **UC-02 — Triage an alert:** understand what fired, where, severity, whether safety
  interlocks already acted; acknowledge; record action taken; resolve or escalate. Alert
  history preserved. (FR-ALR-*)
- **UC-03 — Inspect a zone (rack/tier):** current desired vs actual environment, active
  schedule, current batch, device states, recent telemetry. (FR-DEV-*, FR-TEL-*)

**Configure & control**
- **UC-04 — Commission a device/rack:** physically connect, power on, claim the device
  into the registry, verify advertised capabilities, assign to rack/tier, validate with a
  test command before production use. (FR-DEV-*)
- **UC-05 — Configure a growing zone:** apply a recipe (or manual setpoints) to a tier —
  limited to what the zone's devices can actually do (capability-aware UI). (FR-CTL-*,
  FR-RCP-*)
- **UC-06 — Manual override with expiry:** force an actuator (e.g., lights on, extra
  flood) temporarily; override is visible, time-boxed, audited, and reverts safely.
  (FR-CTL-*)
- **UC-07 — Emergency behavior:** E-stop is hardwired (not a software feature); software
  must *reflect* the E-stop state, stop issuing conflicting commands, and support
  post-incident review. (FR-SAF-*)

**Crop production**
- **UC-08 — Create/edit a crop recipe:** start from a template or existing version;
  edit; save as a new immutable version; mark as template for reuse. (FR-RCP-*)
- **UC-09 — Plan a batch:** choose crop/variety, quantity of trays, seed date; assign
  tier(s); pin a recipe version; see projected harvest date/yield and conflicts.
  (FR-BAT-*)
- **UC-10 — Run a batch through its lifecycle:** seeded → germination/blackout → grow →
  harvest window → harvest; record observations, interventions, issues (with photos
  optional). (FR-BAT-*)
- **UC-11 — Harvest & record yield:** record wet weight, quality grade, waste, tray
  count; system creates an inventory lot and closes batch provenance. (FR-BAT-*,
  FR-INV-*)
- **UC-12 — Adjust a running batch:** swap zone setpoints for this batch only without
  touching the recipe version (deviation recorded against the batch).

**Inventory & sales**
- **UC-13 — Track inventory:** lots, expiry, reserved vs free stock; waste/loss
  recording. (FR-INV-*)
- **UC-14 — Review availability (operator):** what can be promised to customers on a
  date — physical stock + risk-adjusted forecast, minus commitments. (FR-INV-*,
  FR-FCT-*)
- **UC-15 — Place/fulfill an order (P5, portal):** browse, order (incl. custom packs),
  reserve against physical or forecast stock with rules, see delivery date. (FR-ORD-*,
  FR-FCT-*)
- **UC-16 — Reconcile forecast to reality:** after harvest, compare expected vs actual
  yield; feed confidence model. (FR-FCT-*)

**Analysis & data**
- **UC-17 — Answer production questions:** highest-yield recipe? best-performing rack?
  water/energy per kg? which conditions correlate with quality? (FR-ANL-*)
- **UC-18 — One-click export/backup; restore:** complete, versioned, recoverable.
  (FR-BKP-*)
- **UC-19 — Diagnose device issues (P4):** offline devices, stale telemetry, impossible
  sensor values, command failures; firmware update with staged rollout and rollback.
  (FR-DEV-*, FR-SAF-*)

**Build-time**
- **UC-20 — Simulate a facility:** develop and test software (including failure modes)
  against virtual racks/sensors/actuators before — and alongside — physical hardware.
  (FR-SIM-*)

## 5. Success criteria (for the platform as a whole)

- The operator can state facility health from one screen without training (UC-01).
- No single failure of cloud/internet/backend allows an unsafe physical state; every
  actuator's de-energized state matches `safety-rules.json`.
- Any historical batch can be re-derived: which recipe version, which environment
  history, which devices, what happened, what came out.
- Availability shown to a B2B customer never overstates what physics can deliver
  (no oversell without an explicit manual override decision).
- A new device type (e.g., a future aquarium light) integrates without changing core
  schemas — only a new capability profile.