# Requirements — Trophic CEA Platform (foundation)

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** functional requirements (§3) and non-functional requirements (§4)
of the foundation analysis. Each requirement carries an ID, a priority tier, and the
release that delivers it (see [release-plan.md](../roadmap/release-plan.md)). Acceptance
criteria per release are listed there; per-feature criteria are authored in each
pipeline's PRD/L (`docs/pipeline/<slug>/`).

**Separation discipline (working rule 7):** throughout, `Requirement` = a binding statement;
`Assumption` = believed true, unverified — must be confirmed before the requirement it
supports ships; `Recommendation` = default approach, changeable; `Decision` = recorded in
an ADR. Assumptions are marked **[A]** inline and consolidated in the open-questions doc.

## 1. Scope

In scope: the CEA monitoring, control, production, R&D and (later) B2B-sales platform for
Sholaverde's Ooty facility, built by Trophic, deployable to other facilities later.
Out of scope (separate product, shared platform only): aquarium/aquascaping suite UI;
out of scope entirely: open-field agriculture features.

## 2. Hardware reality rule

No functional requirement assumes a specific sensor/actuator exists. Requirements are
written against **capability classes** (device-control-model.md §3); a zone delivers a
requirement only if its devices advertise the matching capability. Hardware capability
status (supported / planned / optional / future) is maintained in
[device-control-model.md](../iot/device-control-model.md) §2, sourced from the BOM and
Rack A engineering docs — the single place hardware truth lives.

## 3. Functional requirements

Priority tiers: **M** = must (release is invalid without it), **S** = should, **C** =
could (explicit backlog).

### 3.1 Facility & zone management (FAC)
- **FR-FAC-01 (M, R1)** — Model the physical hierarchy: facility → room → rack → tier,
  each with identity, attributes, and lifecycle (installed/active/maintenance/retired).
- **FR-FAC-02 (M, R1)** — A tier may be grouped into a **zone** (default: 1 tier = 1
  zone) so environmental configuration targets a logical unit, not a fixed physical one
  (supports future: multi-tier zones, room-level zones).
- **FR-FAC-03 (M, R1)** — Zones have an operational status (active/idle/offline) with
  operator-visible reasons.
- **FR-FAC-04 (S, R1)** — Facility overview aggregates zone states, active batches,
  alarms, connectivity — the Home screen data contract (UC-01).
- **FR-FAC-05 (S, R1)** — Physical metadata: rack position (room layout reference),
  tier height, notes. No CAD dependency — reference links to `doc/facility/` docs only.
- **FR-FAC-06 (C, R6)** — Room-level environmental zones for capabilities that act at
  room scale (HVAC, CO2 enrichment — capability-class `room_*`).

### 3.2 Device registry & management (DEV)
- **FR-DEV-01 (M, R1)** — Device registry: identity, type, hardware/firmware versions,
  claimed/unclaimed state, zone assignment, commissioning date.
- **FR-DEV-02 (M, R1)** — Devices advertise a **capability profile** at boot/provisioning
  (structured, versioned descriptor); the platform never hardcodes per-device behavior.
- **FR-DEV-03 (M, R1)** — Commissioning workflow (UC-04): claim → verify capabilities →
  assign to zone → guided test → activate; every step recorded.
- **FR-DEV-04 (M, R1)** — Device health states: online / offline / stale-telemetry /
  degraded / fault, with last-seen timestamps surfaced.
- **FR-DEV-05 (M, R2)** — Command history per device: issued, ack result, latency,
  failures — visible in device detail.
- **FR-DEV-06 (S, R2)** — Firmware inventory per device + fleet view (P4): versions,
  pending updates, rollback points.
- **FR-DEV-07 (S, R2)** — Sensor calibration metadata: type/model, calibration date,
  due status; calibration expiry raises a maintenance alert. **[A]** (calibration
  practice per sensor class to be confirmed by electronics specialist).
- **FR-DEV-08 (C, R4)** — Device maintenance records (cleaning, descale, replacement).

### 3.3 Telemetry (TEL)
- **FR-TEL-01 (M, R1)** — Ingest telemetry keyed by (zone, device, metric, timestamp)
  with source-clock timestamps preserved; late/out-of-order data handled explicitly.
- **FR-TEL-02 (M, R1)** — Store raw values with per-sample quality flags (good /
  suspect / implausible-rejected); never silently discard (device-control-model §6).
- **FR-TEL-03 (M, R1)** — Compute and store **calculated** series (VPD from T/RH; DLI
  from light state/photoperiod) with formula version recorded.
- **FR-TEL-04 (M, R1)** — Staleness policy: each zone shows data age; staleness beyond
  thresholds raises OPERATIONAL alert.
- **FR-TEL-05 (S, R1)** — Downsample/rollup strategy defined from the start
  (raw → 1m → 1h rollups; data-architecture.md).
- **FR-TEL-06 (S, R4)** — Resource metering ingestion (water per flood/flow, kWh)
  for zones with metering capability; estimates clearly marked when metering is absent.
- **FR-TEL-07 (C, R6)** — Image capture storage linked to batch timeline (if/when
  imaging hardware exists).

### 3.4 Control & commands (CTL)
- **FR-CTL-01 (M, R2)** — Desired-state store per zone per capability, with full
  history (who/what/when set it) — the operator-visible "Desired".
- **FR-CTL-02 (M, R2)** — Schedule execution at the **edge controller** (photoperiod,
  flood-and-drain cycles, fan programs) — must run without cloud connectivity.
- **FR-CTL-03 (M, R2)** — Command pipeline with: idempotent command IDs, ack/reject with
  reason, bounded retries, timeout expiry, and visibility of the whole lifecycle (UC-03).
- **FR-CTL-04 (M, R2)** — Manual override: per-actuator, time-boxed (auto-revert),
  confirmation UI showing safety envelope, audible/visible banner while active, full
  audit.
- **FR-CTL-05 (M, R2)** — Safety-envelope enforcement at the edge: commands/config
  outside sourced bounds (`safety-rules.json`) are rejected regardless of origin;
  rejection reason is machine-readable and surfaced.
- **FR-CTL-06 (M, R2)** — Capability-aware control UI: an operator can only be offered
  controls a zone's devices actually support.
- **FR-CTL-07 (M, R2)** — Emergency-state reflection: E-stop engaged / interlock tripped
  states are displayed prominently; while active, conflicting commands are refused with
  the reason shown.
- **FR-CTL-08 (S, R7)** — Closed-loop regulation (e.g., VPD-derived humidity control,
  CO2 dosing interlocked with ventilation) — only where capability + sourced targets
  exist; each loop individually gated by domain review.
- **FR-CTL-09 (C, R7)** — Control-loop policy versioning + per-loop enable flags +
  loop health metrics (oscillation detection, actuator duty monitoring).

### 3.5 Alerts (ALR)
- **FR-ALR-01 (M, R1)** — Rule-based alerts from telemetry/desired-state comparison:
  threshold + duration semantics, per-zone configurable, severity tiers SAFETY /
  OPERATIONAL / ADVISORY.
- **FR-ALR-02 (M, R1)** — Alert lifecycle: raise → notify → acknowledge (with reason
  for SAFETY) → resolve; full history retained and queryable.
- **FR-ALR-03 (M, R1)** — SAFETY alerts cannot be dismissed without a recorded action
  and are visually dominant on every screen while active.
- **FR-ALR-04 (S, R1)** — Notification channels: console + push/email (operator);
  channel per severity configurable.
- **FR-ALR-05 (S, R2)** — Device-failure alerts: offline, stale, command-failure
  patterns, impossible-value patterns (sensor fault inference).
- **FR-ALR-06 (S, R1)** — Every alert links to its rule definition and the
  threshold's source citation (safety-rules.json where applicable).

### 3.6 Recipes & templates (RCP)
- **FR-RCP-01 (M, R3)** — Recipe entities grouped by capability domain, with
  crop/variety association and template flagging.
- **FR-RCP-02 (M, R3)** — Immutable versioning: versions never change once referenced;
  lineage (cloned-from) preserved; content hash stored (production-model §4).
- **FR-RCP-03 (M, R3)** — Save-as workflow: template → applied instance → edited →
  new version (never writes back to source).
- **FR-RCP-04 (M, R3)** — Recipe validation at save AND apply: safety envelope (sourced
  bounds), completeness vs. target zone capability profile.
- **FR-RCP-05 (S, R3)** — Lifecycle parameters: germination/blackout days, grow days,
  harvest window, expected yield/date — advisory, per-version.
- **FR-RCP-06 (S, R3)** — Import/export of recipes as versioned artifacts (part of
  backup bundle).

### 3.7 Batches & production (BAT)
- **FR-BAT-01 (M, R3)** — Batch lifecycle per production-model §3 (PLANNED → SEEDED →
  GERMINATION → GROWING → HARVEST_WINDOW → HARVESTED; ABORTED/CANCELLED with reasons).
- **FR-BAT-02 (M, R3)** — Batch pins recipe version + content hash; zone assignment
  records (multi-assignment history).
- **FR-BAT-03 (M, R3)** — Batch event log: observations, interventions, issues (with
  severity), photos (optional) — append-only with actor/timestamp.
- **FR-BAT-04 (M, R3)** — Harvest recording: weights, grade, waste, tray count; creates
  inventory lots; records actual-vs-expected delta.
- **FR-BAT-05 (S, R3)** — Production calendar view: batches by day, planned vs actual.
- **FR-BAT-06 (S, R3)** — Experimental batches flagged, excluded from operational
  aggregates, included (tagged) in R&D queries.

### 3.8 Inventory (INV)
- **FR-INV-01 (M, R4)** — Inventory lots with provenance, grade, best-before, location,
  loss events.
- **FR-INV-02 (M, R4)** — Stock states per inventory-model §2: physical / reserved /
  available / expired; operator-facing availability timeline.
- **FR-INV-03 (M, R4)** — Reservation management: against lots and against future
  batch harvests; atomic netting; abort-impact alerting (which commitments broke).
- **FR-INV-04 (S, R4)** — Manual stock entry with mandatory reason code (audited).

### 3.9 Forecast availability (FCT)
- **FR-FCT-01 (M, R4)** — Per-batch HarvestForecast: expected date/yield with
  confidence mechanism per inventory-model §3 (parameters domain-reviewed before R5;
  marked provisional).
- **FR-FCT-02 (M, R4)** — Forecast revision history (estimates are never silently
  overwritten — rd-data-model §2).
- **FR-FCT-03 (M, R4)** — Availability computation = available stock + risk-adjusted
  forecast − commitments; every customer-visible number derives from this, never raw
  expected yield.

### 3.10 B2B portal & orders (ORD)
- **FR-ORD-01 (M, R5)** — Customer accounts (separate identity domain), browse catalog +
  availability by date, place orders incl. custom packs.
- **FR-ORD-02 (M, R5)** — Ordering honors forecast confidence floor; over-request
  beyond availability is impossible by construction.
- **FR-ORD-03 (M, R5)** — Order states + cancellation windows per inventory-model §4;
  customer-visible status.
- **FR-ORD-04 (M, R5)** — Order commitment exceptions (shortfall from failed/short
  harvests) surface as operator workflow, not silent re-promising.
- **FR-ORD-05 (S, R5)** — Recurring availability view (customer sees typical cadence
  by crop — derived from production history).
- **FR-ORD-06 (C, R6+)** — Invoicing/payment integrations — explicitly deferred.

### 3.11 Analytics / R&D (ANL)
- **FR-ANL-01 (M, R6)** — Query layer over batch × recipe-version × zone-history ×
  harvest joins (the questions in rd-data-model §4 without custom code).
- **FR-ANL-02 (S, R6)** — Resource-per-kg analytics (water, electricity, cost) with
  measured/estimated provenance always displayed.
- **FR-ANL-03 (S, R6)** — Saved/repeatable reports; export (CSV/JSON) of any query.
- **FR-ANL-04 (C, R8)** — Predictive yield modeling — only after data quality is
  demonstrated (working rule 17).

### 3.12 Backup / export / restore (BKP)
- **FR-BKP-01 (M, R1)** — One-click full export: versioned, self-describing bundle
  (schema version, content manifest, checksums) covering config, recipes + versions,
  batches, inventory, orders, users/roles, audit, telemetry summaries (raw telemetry
  archived separately per retention policy).
- **FR-BKP-02 (M, R1)** — Restore/import procedure with validation (schema migration,
  checksum verify, conflict policy: import as new vs. replace — operator chooses with
  warnings).
- **FR-BKP-03 (S, R1)** — Scheduled automatic backups with retention, health alerts on
  failure, restore drill instructions. **[A]** (host storage target to be decided at
  deploy time).
- **FR-BKP-04 (S, R1)** — Selective export (e.g., R&D dataset) without full backup.

### 3.13 Simulation (SIM)
- **FR-SIM-01 (M, R1)** — Virtual device support: the same device/capability model runs
  against simulated hardware, so registry, telemetry, commands, dashboards are all
  testable without physical racks.
- **FR-SIM-02 (S, R2)** — Fault injection: offline, stale, implausible values, command
  failures, controller reboot.
- **FR-SIM-03 (C, R6)** — Environment simulation (thermal/VPD/humidity dynamics) for
  algorithm testing — deliberately deferred; physics fidelity is its own R&D problem.

### 3.14 Platform & users (USR)
- **FR-USR-01 (M, R1)** — AuthN/AuthZ per security-architecture §2 (RBAC roles,
  TOTP); all operator actions audit-logged.
- **FR-USR-02 (M, R1)** — Append-only audit log (security-architecture §5).
- **FR-USR-03 (S, R1)** — Platform self-observability: service health, broker status,
  ingest lag, storage growth — surfaced in Settings/maintenance, alertable.
- **FR-USR-04 (S, R1)** — org-scoping of all core entities (ADR-0005).

## 4. Non-functional requirements

### Reliability & availability (the control system)
- **NFR-REL-01 (M)** — Safe operation with **zero** cloud connectivity: schedules,
  interlock-adjacent logic, and safety-envelope enforcement run at the edge
  indefinitely; cloud restores as a client (ADR-0001).
- **NFR-REL-02 (M)** — Backend outage: no unsafe state change is possible; telemetry
  buffers at the edge and syncs on restore with explicit gap marking.
- **NFR-REL-03 (M)** — Power restoration: devices boot to documented fail-safe states
  (`safety-rules.json`), controller resumes schedules only after self-check; no command
  replay storms.
- **NFR-REL-04 (M)** — Every actuator class honors its documented de-energized state —
  this is verified at commissioning and re-verified in firmware updates (60s leak
  interlock proof is an existing QC hold point).
- **NFR-REL-05 (S)** — Edge controller uptime target ≥99.5% monthly; watchdog reset
  recovers to safe schedule resumption.

### Performance & scale (current facility sized, headroom-designed)
- **NFR-PER-01 (M)** — Ingest ≥50 zones × typical sensor cadence (30–60 s) + event
  bursts without loss; **[A]** cadence confirmed per sensor class in LLD.
- **NFR-PER-02 (M)** — Console Home loads with facility state in <2 s on a normal
  connection; tier drill-down <3 s.
- **NFR-PER-03 (S)** — Command round-trip (console→controller→ack visible) <2 s on LAN;
  remote (via backend) <5 s typical.
- **NFR-PER-04 (S)** — Design headroom: 1 room → ≥4 rooms, ≥100 zones, no schema
  redesign (measured against the 11-rack / 44-zone current floor plan).

### Data integrity & retention
- **NFR-DAT-01 (M)** — Historical batch provenance is never mutable: recipe versions
  immutable, assignments/phase transitions append-only, corrections are new records.
- **NFR-DAT-02 (M)** — Telemetry write-path durability: acknowledged writes survive
  process restart; no silent gaps (gaps are recorded gaps).
- **NFR-DAT-03 (S)** — Retention: batch/production/audit data indefinitely; raw
  telemetry ≥24 months then rollups retained indefinitely; exact policy tunable at ops
  level without data loss (raw exportable before rollup).
- **NFR-DAT-04 (M)** — Backup RPO ≤24 h (scheduled) / on-demand 0; RTO ≤4 h for full
  stack on existing infra. **Recommendation**, pending ops confirmation of host.

### Security & safety compliance
- **NFR-SEC-01 (M)** — security-architecture.md is normative; `security-safety-reviewer`
  gates every safety-relevant change (never skipped — pipeline rule).
- **NFR-SAF-01 (M)** — No software component may create or require an operating state
  that contradicts `safety-rules.json`; unknown thresholds block, never guess
  (workspace law, repeated here as an NFR because it binds the product, not just agents).
- **NFR-SAF-02 (M)** — TBD thresholds (EC operating target; water-recovery gate G1) are
  hard blockers for the features that would consume them; tracking in open-questions doc.

### Usability
- **NFR-UX-01 (M)** — uc IA principles 1–7 are requirements, not suggestions (safety
  visibility, desired-vs-actual, staleness display, mobile-first operator flows).
- **NFR-UX-02 (M)** — A new operator reaches UC-01/UC-03 competence with ≤30 min
  orientation; validation in e2e (R1 acceptance).
- **NFR-UX-03 (S)** — All timestamps displayed in facility-local time + UTC available;
  units SI (°C, %, µmol/m²/s, kg) consistent per persona locale defaults.

### Maintainability & operability
- **NFR-OPE-01 (M)** — Deployment via the existing docker-compose stack; config via
  environment + admin UI, no file surgery for routine ops.
- **NFR-OPE-02 (M)** — Platform observability: ingest lag, broker health, disk growth,
  service restarts visible + alertable (FR-USR-03) — the maintainer persona needs this
  before telemetry volume grows.
- **NFR-MNT-01 (M)** — Monorepo, pipeline-gated (existing `/ship` `/extend` `/fix`),
  every release's acceptance criteria defined before build (working rule 20).

### Localization / deployment environment
- **NFR-ENV-01 (S)** — Runs on a single facility host (current design) and, later, a
  cloud VM without code change — the only difference is which side of the NAT the B2B
  portal reads through; VPN/relay decision at R5. **[A]**
- **NFR-ENV-02 (S)** — India-specifics: INR display, IST/GMT+5:30, power-failure
  resilience assumptions (generator/UPS per Phase A §2.4) documented in ops runbook.

## 5. Traceability

Requirement → release mapping is per-ID above; release-level acceptance criteria live in
[release-plan.md](../roadmap/release-plan.md); open questions blocking requirements are
in [open-questions-risks-next-actions.md](../decisions/open-questions-risks-next-actions.md).