# 02 — Product strategy note / PRD: Phase 1 (Release 1) foundation

**Pipeline:** `/ship` · **Slug:** `phase-1-foundation` · **Date:** 2026-09-10
**Author:** product-strategy-specialist · **Consumes:** `01-classification.md`
**Scope authority:** `docs/roadmap/release-plan.md` §"Phase 1 (Release 1)" — **LOCKED by the
owner 2026-09-08, un-paused 2026-09-10 to run on the simulator.** This note does not
re-scope. It restates the locked scope as testable acceptance criteria, records where the
requirements catalogue and the lock disagree (lock wins), states what is out, and defines
"done" for the six validation items the lock names.

**Roadmap / portfolio fit (one line):** **Fits as planned** — this is the Phase E Year-1
"launch Product 1.6" software layer (Phase E TRL table: 1.6 is TRL 6-7 precisely because
"the cloud dashboard/alerting software layer needs building") and the spine Product 2.1
(Year 2) is scheduled to sit on; no deviation from the 3-year roadmap or the Phase C
portfolio. See §8 for the one framing nuance (Phase C says "cloud dashboard"; ADR-0001/0005
deliver an on-facility platform with cloud optional — an already-accepted stance, not a
Phase 1 deviation).

---

## 1. Problem statement

Nothing runs. `backend/`, `frontend/`, `firmware/` contain `.gitkeep`; `docker-compose.prod.yml`
is fully commented out; the accepted ADRs (0001, 0002, 0004, 0005, 0006, 0008, 0009)
constrain a platform that does not yet exist. Every later release (control R2, recipes and
batches R3, inventory R4, portal R5, R&D R6, loops R7) reuses the same seams — identity,
capability model, transport, telemetry store, alert lifecycle, backup, deploy, simulation —
so the semantics fixed here are inherited, not revisited. The specific risks Phase 1 exists
to retire, in priority order:

1. **Semantics that are expensive to retrofit** — org-scoping on every query (ADR-0005
   names this as the R1 acceptance criterion), telemetry quality flags that never drop
   (device-control-model §6), alert severity tiers with an undismissable SAFETY class, an
   append-only audit log. Getting any of these wrong is a design flaw R2+ inherits
   (classification §"Safety relevance").
2. **A hardware-independent build loop** — the room controller compute platform is
   undecided (OQ-3, closing as ADR-0007 in parallel); the software spine must be provable on
   a virtual 11-rack facility so no deliverable waits on hardware (system-architecture §7,
   owner decision 2026-09-10).
3. **The first operator-visible value** — Home answers "is anything wrong?" for P1 without
   the platform ever being able to actuate anything, so the physical risk of this release is
   near zero while the platform's reason to exist becomes visible (release-plan "Why this is
   the right first slice").

## 2. Target users and use-cases

| Persona | Phase 1 surface | Use-cases served | What they cannot do yet |
|---|---|---|---|
| **P1 Operator/Maintainer** (primary) | Home, Alerts, Grow (read) | UC-01 glance, UC-02 triage (ack/resolve), UC-03 inspect zone (telemetry only) | No desired-vs-actual (R2), no override (R2), no batches (R3) |
| **P4 Device technician** | Devices, commissioning flow | UC-04 commission (claim → verify → assign → identify-class test → activate), UC-19 diagnose (offline/stale/implausible) | No firmware/OTA (R2+), no command history view (FR-DEV-05, R2), LAN only — remote path is OQ-12/R5 |
| **P2 Grow manager / P3 owner** | Console, read-mostly (viewer/grower roles) | Facility state, alert history, export | No recipes, production, analytics |
| **Admin** (likely P1/P2 same person) | Settings | Users/roles, backup/export/restore, self-observability | No org/tenant admin (ADR-0005 item 3) |
| **Build team / CI** (UC-20) | Sim harness, virtual devices | Run the whole stack with zero hardware; fault switches | Command failures, reboot, physics (FR-SIM-02 remainder, FR-SIM-03) |

P5 (B2B), P6 (external tenant), P7 (aquarium) have no Phase 1 surface.

## 3. Scope

### 3.1 In scope — the locked FR/NFR list

FR-FAC-01..05 · FR-DEV-01..04 · FR-TEL-01..05 · FR-ALR-01..04 + 06 · FR-SIM-01 (v0: virtual
rack + virtual room controller) **+ the three fault switches** (see 3.2) · FR-USR-01..04 ·
FR-BKP-01..02 (+03 basic scheduled) · NFR-REL-01/02 *architecture shape only* · NFR-PER-01/02
· ADR-0005 tenant-scoping test suite. Concrete deliverables 1–5 as written in the release
plan (Go service + Postgres/Timescale + Mosquitto in compose; console v0 with 5 screens;
firmware v0 skeleton + virtual profiles; sim harness v0; deploy + backup job + restore
drill).

### 3.2 Drift between `requirements.md` and the lock — lock wins, recorded here

| Item | `requirements.md` says | Locked plan says | Disposition |
|---|---|---|---|
| Fault switches offline / stale / implausible | FR-SIM-02 (S, **R2**) | Deliverable 4: "1 virtual rack (4 tiers) + fault switches (offline, stale, implausible)" | **In scope** — these three switches only. The remainder of FR-SIM-02 (command failures, controller reboot) stays R2. FR-SIM-02's ID is carried as a partial. `requirements.md` should be annotated "partially delivered R1" by the maintainer loop — not edited by this pipeline. |
| FR-BKP-04 selective export | S, **R1** | Not in the locked ID list | **Out** of Phase 1. Next release picks it up. |
| NFR-UX-02 ≤30 min orientation, "validation in e2e (R1 acceptance)" | M, R1 acceptance | Not in the locked list | **Not a Phase 1 done-gate.** qa-e2e-validator may run it informally on the 5 screens; a fail is a finding, not a block. |
| NFR-DAT-02 telemetry durability, "no silent gaps" | M, no release tag | Realised by the 72 h soak criterion | **Effectively in** — the soak's "zero gaps unaccounted" is this NFR; noted so traceability is explicit. |
| NFR-UX-01 IA principles 1–7 "are requirements" | M, no release tag | Lock: "every number shows source age" | Principles **1–4** bind the five Phase 1 screens (safety visibility, glance→drill→act, desired-vs-actual *where desired exists*, offline-tolerant/age display). 5–7 are surface-level and hold trivially. |

### 3.3 Explicitly out of scope (Phase 1 cannot do these; tests must not expect them)

- **Any actuation.** No command to a real or virtual valve, LED, fan, light, pump, doser, or
  HVAC. No desired-state store (FR-CTL-01), no schedules (FR-CTL-02), no command pipeline
  beyond a single identify-class ack (FR-CTL-03), no overrides (FR-CTL-04), no envelope
  enforcement (FR-CTL-05), no control UI (FR-CTL-06), no E-stop reflection (FR-CTL-07).
  The backend has **no code path that publishes to any `cmd` topic except the identify /
  no-op class** used by commissioning (AC-DEV-03e).
- **Recipes** (FR-RCP-*, ADR-0003 untouched), **batches** (FR-BAT-*), **inventory /
  forecast** (FR-INV-*, FR-FCT-*), **portal / orders** (FR-ORD-*), **analytics** (FR-ANL-*).
- **Closed loops** (FR-CTL-08/09, R7). **OTA / firmware inventory** (FR-DEV-06, security
  §6). **Command history view** (FR-DEV-05). **Calibration metadata and expiry alerts**
  (FR-DEV-07). **Device-failure alert inference** (FR-ALR-05 — offline is a *health state*
  and a Home tile in Phase 1, not an inferred alert; see 3.4-c).
- **Room-level environmental zones** (FR-FAC-06), **selective export** (FR-BKP-04),
  **environment physics simulation** (FR-SIM-03), **metering** (FR-TEL-06), **imaging**
  (FR-TEL-07).
- **Real room-controller firmware** and its local UI / local TSDB (OQ-3 → ADR-0007; not
  re-litigated here). **Edge telemetry buffering and backfill from the room controller's
  local store** (ADR-0001 point 4) — see 3.4-e.
- **Multi-tenant anything** (org switching, tenant admin, billing, per-tenant keys —
  ADR-0005 item 3). **Remote access / VPN / cloud endpoint** (OQ-12, R5); Phase 1 is LAN
  only. **Per-PR integration compose stack** (`docs/pipeline/README.md` Phase 2 CI/CD).
- **A browser-side MQTT client of any kind** (ADR-0009 explicit boundary). **PWA caching of
  API data** (ADR-0009: shell only).
- **Aquarium/aquascaping** surfaces (IA principle 6, ADR-0008 T2). **Saffron** anything
  (OQ-1 closed).
- **Per-tier climate control, per-tier CO₂, per-tray level, fixed PPFD sensing** — absent
  by design (`trophic-contracts/capabilities/absent-by-design.md`); no virtual profile
  advertises them, no screen renders them.
- **Hardware.** No PCB, pinout, enclosure, or physical commissioning. "≥1 real rack
  controller if hardware ready" in the lock is optional and does not gate done.

### 3.4 Scoping clarifications forced by "no actuation" (interpretations for the HLD to confirm)

These are readings of the lock, not additions. Each is flagged because a downstream
specialist could reasonably read it the other way.

- **(a) Alert rules compare telemetry to rule thresholds, not to desired state.** FR-ALR-01
  says "telemetry/desired-state comparison"; there is no desired state until R2. Phase 1
  rules are `metric × comparator × threshold × duration × hysteresis × zone scope ×
  severity` (data-architecture §3), with the threshold living on the rule. "Off-target vs
  desired" is R2.
- **(b) Home is a subset of IA §3 Home.** Tiles that exist: active alarms (by severity, ack
  state), zones by status, device issues (offline/stale/degraded/fault), connectivity and
  last-sync. Tiles that **do not render** in Phase 1: "batches due" (R3) and "zones
  off-target (desired vs actual)" (R2). They are omitted, not shown as empty zeros — the UI
  analogue of "a control must not appear for a capability the hardware does not have".
- **(c) Zone status active/idle (FR-FAC-03)** has no batch or schedule input to derive
  from. In Phase 1 `active`/`idle` is **operator-set with a reason** (audited); `offline` is
  **derived** from device health. R2/R3 make active/idle derived.
- **(d) The commissioning "guided test" is an identify-class, non-actuating command.** IA
  J6's example ("LED on 10% — operator confirms visually") is the R2 form. The capability
  vocabulary (device-control-model §3) has no such class today; the HLD must add one
  (e.g. `control.identify`) — an ADR-0002 vocabulary registration, not a scope change.
- **(e) NFR-REL-02 "architecture shape"** is read as: (1) the backend never becomes
  something the room depends on — stopping it must not stop the virtual room controller or
  virtual racks; (2) **sequence-gap detection and gap *recording* are required** (the soak
  criterion needs them); (3) **edge buffering and sequence-verified backfill from the room
  controller's local store are R2** (they need the real room controller, OQ-3). The HLD
  should say so explicitly.
- **(f) Virtual device profiles advertise the true Rack A capability set** (fill/drain
  `valve.control`, `irrigation.flood`, `light.dim`, `airflow.speed` and the sensing
  classes), so R2 needs no new profiles; Phase 1 *exercises* only sensing, events, and
  identify. The "no actuation" guarantee lives in the backend (no `cmd` path) and the sim
  (non-identify commands → `REJECTED: not implemented in v0`), not in a pruned profile.
  Recommendation, not a requirement — the HLD may choose a narrower v0 profile if it
  documents the R2 upgrade path.
- **(g) DLI (FR-TEL-03)** needs canopy PPFD, which the hardware cannot sense in service
  (absent-by-design assertion 2). In Phase 1 the DLI series exists as a *provenance-marked
  estimate* computed from `light.state_report` × a registry-recorded commissioning PPFD
  value. There is no schedule to drive real light state, so the acceptance test runs
  against virtual `light.state_report`. VPD is the primary proof of the calculated-series
  mechanism.
- **(h) Alerts that exist in Phase 1:** staleness (FR-TEL-04, mandatory OPERATIONAL),
  telemetry-threshold rules (FR-ALR-01), platform self-observability (FR-USR-03: at least
  broker-disconnected), backup-failure (FR-BKP-03). Device *offline* surfaces as a health
  state and Home tile; raising it as an alert is FR-ALR-05 (R2) and not required.

## 4. The virtual facility (sourced shape — the fixture every acceptance test runs against)

All facts below are from the accepted docs / trophic-contracts v0.2.0; nothing is invented.

| Element | Value | Source |
|---|---|---|
| Racks | 11 (parametrisable; sim v0 default fixture is 1 rack, soak fixture is 11) | release-plan deliverable 4 and validation; instrument-tags "11 x ESP32 rack controllers" |
| Tiers per rack | 4 (Rack A form factor; 3–5 supported) | workspace-audit §3.2-7; safety-rules `flood_and_drain_timing` |
| Zones | 44 by default (1 tier = 1 zone, FR-FAC-02) | NFR-PER-04 "11-rack / 44-zone" |
| Rack-level sensing | 1 × T/RH per rack at tier 3 (standard build; per-tier is OPTIONAL) | absent-by-design; device-control-model §2 |
| Per-tier telemetry | fan tacho (`airflow.tacho`, "alarm not optional"), `light.state_report` | instrument-tags per-rack table; device-control-model §3 |
| Per-rack events | leak puck `LD-01..LD-11`, header high switch `LSH-04` (hard-wired; software *reads* state) | instrument-tags; device-control-model §2 |
| Room-level (virtual room controller) | CO₂ × 2 NDIR (control @1.5 m, safety @300 mm), `TE-01`/`TE-02` water temp, `AT-03` EC, `AT-04` pH, `LT-02` level, `FS-01` flow | instrument-tags; device-control-model §2 |
| Controller roles | `control.compute` = rack (×11) / room (×1); rack controllers are **Modbus nodes, not MQTT clients**; the room controller is the **sole edge MQTT client** | ADR-0004 point 1; instrument-tags |
| Topic taxonomy | `trophic/<org>/<site>/<room|rack>/<tier?>/<device>/{telemetry/<metric>, state/reported, state/desired, cmd, ack, event}` with the stated QoS/retain flags | device-control-model §4 (the `trophic/` root is unrelated to the GitHub org, CLAUDE.md) |
| Units | °C, % RH, kPa (VPD), ppm (CO₂ alarm) / µmol/mol (CO₂ target), mS/cm (EC), µmol/m²/s (PPFD), m³/h (airflow), mm (lengths) | `conventions/units.md` |
| Sample schema | `{device_id, metric, value, unit, ts_source, seq, quality, calibration_id}` | device-control-model §6; data-architecture §3 |
| Cadence | 30–60 s "typical sensor cadence" — **[A] confirm per class in LLD** | NFR-PER-01 |

The virtual room controller v0 is an MQTT client that *fronts* the virtual racks; it does
not need a local TSDB, local UI, drain-token, or reuse-gate logic in Phase 1 (those are
actuation/process — R2, system-architecture §7).

## 5. Acceptance criteria per FR ID

Conventions: each AC is a statement `qa-e2e-validator` can turn into a pass/fail check.
"Sim" = the harness with the virtual facility of §4. "Audited" = an append-only audit row
with actor, timestamp, before/after where applicable (security-architecture §5). Where a
number is not in a source document it is marked **[unsourced — LLD/qa to fix]** and is not a
done-gate on its own.

### 5.1 Facility & zones (FAC)

| ID | Acceptance criterion |
|---|---|
| AC-FAC-01a | Facility → Room → Rack → Tier exist as entities with identity, attributes, and lifecycle `installed \| active \| maintenance \| retired`; every lifecycle transition is audited with actor. |
| AC-FAC-01b | The 11-rack / 44-tier virtual facility can be created via API or seed and is rendered in Grow as a drillable hierarchy (facility → room → rack → tier). |
| AC-FAC-02a | Creating a tier auto-creates a 1:1 zone. The schema permits a zone containing >1 tier (storable via API); no multi-tier zone UI is required. |
| AC-FAC-03a | Zone status `active \| idle \| offline` with an operator-visible reason. `offline` is derived: when all of a zone's devices are `offline` or `stale-telemetry`, the zone shows `offline` and the reason names the device(s). `active`/`idle` are operator-set with reason in Phase 1 (3.4-c), audited. |
| AC-FAC-03b | Sim `offline` switch on a virtual rack → its 4 zones show `offline` within the configured staleness window, reason naming the rack controller. Clearing the switch restores the previous operator-set state. |
| AC-FAC-04a | A Home data contract (one endpoint or a bounded documented set) returns: alerts by severity and ack state, zones by status, device issues by health state, broker connectivity + facility-wide last-telemetry age + ingest lag; every value carries an as-of timestamp. Fields for batches/off-target are absent (3.4-b). |
| AC-FAC-05a | Rack position (row/slot label), tier height in mm, free-text notes, and reference links to `doc/facility/` paths are editable by admin and audited. **No numeric X/Y rack coordinates are seeded from the OCR'd floor-plan table** (OQ-4: column misalignment) unless visually verified and cited. |

### 5.2 Device registry (DEV)

| ID | Acceptance criterion |
|---|---|
| AC-DEV-01a | Registry row per device: id, type (descriptive only — ADR-0002 point 4), hardware version, firmware version, `claimed \| unclaimed`, zone assignment, commissioning date, `sim` flag, hardware fingerprint (security §2). Devices screen lists all rows with health state and last-seen. |
| AC-DEV-01b | A virtual device's boot `event` creates an `unclaimed` registry row without operator action. A device reappearing with a mismatched fingerprint is quarantined (flagged, not merged) — security §2. |
| AC-DEV-02a | Capability profile is a schema-versioned JSON descriptor validated at provisioning; the extracted capability index (data-architecture §4) makes "which devices advertise `sense.co2`" a plain query that returns exactly the virtual room controller. |
| AC-DEV-02b | A profile containing an unknown capability class **and** a newer profile schema version is accepted; the unknown class is shown as "unrecognised" in device detail; nothing crashes and nothing is silently configured (ADR-0002 point 3). |
| AC-DEV-02c | Shipped virtual profiles for the standard Rack A build do **not** advertise per-tray level, per-tier CO₂, fixed PPFD, or per-tier climate control (absent-by-design); rack T/RH is advertised once per rack. |
| AC-DEV-03a | Commissioning flow steps, in order, each audited with actor: claim → verify capabilities (profile displayed matches advertised) → assign to rack/tier → guided test → activate. Steps cannot be skipped via the API. |
| AC-DEV-03b | Only `technician` and `admin` roles can claim/assign/activate (security §2 RBAC table); `operator` and `viewer` receive 403 on those endpoints. |
| AC-DEV-03c | An `unclaimed` device may advertise capabilities but is wired to no zone and **receives no commands** (a guided-test request against an unclaimed device is refused server-side). |
| AC-DEV-03d | Guided test = an identify-class, non-actuating command (3.4-d). The ack (`SUCCESS`/`REJECTED` + reason + latency) is visible in the flow; a `REJECTED` or timeout blocks activation until re-run. |
| AC-DEV-03e | Static/contract check: the backend publishes to no `cmd` topic for any class other than the identify class. (This is the "no actuation" guarantee — carried into the security review.) |
| AC-DEV-04a | Health states `online \| offline \| stale-telemetry \| degraded \| fault` exist in the model with last-seen timestamps surfaced on Devices and in device detail. |
| AC-DEV-04b | Sim `offline` → `offline` (via broker disconnect/LWT or event) within the configured window; sim `stale` (connected, no telemetry) → `stale-telemetry` within the per-class max-age; clearing → `online`. `degraded`/`fault` transitions are verified by unit tests on the state machine (the three v0 switches do not drive them). |

### 5.3 Telemetry (TEL)

| ID | Acceptance criterion |
|---|---|
| AC-TEL-01a | Samples are stored keyed by (org, zone, device, metric, `ts_source`) with `seq`; `ts_source` (device clock) is preserved alongside receipt time. |
| AC-TEL-01b | Out-of-order and late samples are stored as received and ordered by `ts_source` on read — never dropped, never re-stamped. A `seq` discontinuity per device creates a **gap record** (§6.1 definition). Contiguous `seq` with out-of-order `ts_source` creates no gap record. |
| AC-TEL-02a | Every stored sample carries `quality ∈ {good, suspect, implausible-rejected}`; implausible samples are stored with the flag, excluded by default from last-known-state, rollups, and alert evaluation, and visible in device health / device detail. |
| AC-TEL-02b | Plausibility bounds are traceable: either the device profile's declared sensor range or a documented physical range (device-control-model §6 examples: RH 0–100 %, T −40..85 °C class, CO₂ 0–10 000 ppm) — **never a crop target**. Sim `implausible` switch emits values outside the bound and every such sample is flagged, none dropped, none reach Home. |
| AC-TEL-03a | VPD is computed per T/RH sample pair and stored as a calculated series whose metadata records formula name + version; recomputing from the stored T/RH with that formula reproduces the stored value. Formula choice is an LLD decision reviewed by botany-horticulture-specialist; the PRD requires only versioning and reproducibility. |
| AC-TEL-03b | DLI series exists per tier, computed from `light.state_report` × a registry-recorded commissioning PPFD attribute, formula-versioned, and provenance-marked **estimate** everywhere it is rendered (3.4-g). Test fixture PPFD is a value inside the sourced 150–210 µmol/m²/s band; it is a fixture, not a claim. |
| AC-TEL-04a | Each sensor class has a configurable max-age; every zone/tier view shows data age; age exceeded → OPERATIONAL alert that resolves when fresh data resumes (ack history retained). Defaults adopted from device-control-model §6 examples (T/RH 2 min, CO₂ 5 min) are **example-grade — LLD confirms per class**. |
| AC-TEL-05a | Continuous aggregates at 1 m and 1 h (data-architecture §3 also names 1 d) exist and are complete over the soak window; tier charts read aggregates for ranges beyond a documented cut-over, raw below it. Raw retention policy is defined and ops-configurable (NFR-DAT-03 target: raw ≥24 months) with export-before-rollup possible. |

### 5.4 Alerts (ALR)

| ID | Acceptance criterion |
|---|---|
| AC-ALR-01a | Rule model: metric, comparator, threshold, duration, hysteresis, zone scope, severity `SAFETY \| OPERATIONAL \| ADVISORY`; thresholds are overridable per zone. Phase 1 compares telemetry to the rule's threshold (3.4-a). |
| AC-ALR-01b | Evaluation is a persisted state machine per rule per zone: a breach shorter than `duration` raises nothing; longer raises once; a backend restart mid-condition does not re-raise (data-architecture §3). |
| AC-ALR-02a | Lifecycle `raise → notify → acknowledge → resolve`, each with actor/timestamp; SAFETY acknowledgement requires a reason string; history is queryable by zone, severity, rule, time range. |
| AC-ALR-03a | An active SAFETY alert is visually dominant and reachable in one tap on **all five** Phase 1 screens (IA principle 1) and cannot be dismissed without a recorded action — verified by navigating every screen with a sim-driven SAFETY condition active. |
| AC-ALR-04a | Channels: console (Home + Alerts centre) always; plus at least one of email / push, configurable per severity in Settings; delivery proven in e2e against a test mail sink (or push equivalent). |
| AC-ALR-06a | Every alert links to its rule and to the threshold's source citation. Any seeded rule that cites a physical threshold copies it **verbatim** from `safety-rules.json` (key path + `source` string). If the CO₂ 5000 ppm rule is seeded it is labelled "mirror of the independent hardware monitor (fail-closed solenoid) — not the safeguard" (classification §Safety 2). An operator-created threshold with no source is labelled "operator-configured, unsourced". No rule may carry a threshold from a `TBD` entry (EC operating target, G1). |

### 5.5 Simulation (SIM)

| ID | Acceptance criterion |
|---|---|
| AC-SIM-01a | Virtual rack (4 tiers) + virtual room controller implement the same capability-profile schema and MQTT topic/payload schema v1 as real hardware; the only distinguishing marker is `sim: true` (device-control-model §10). Registry, ingest, Home, Devices cannot tell them apart in tests. |
| AC-SIM-01b | Harness is parametrisable: 1 rack (dev default), 11 racks (soak), ≥50 zones (NFR-PER-01 load). Runs in CI with zero hardware. |
| AC-SIM-01c | Placement honours ADR-0008: the harness imports nothing from `backend/` or `firmware/` source; it couples via the MQTT schema only. Location (own top-level `sim/` per system-architecture §2, or a virtual target under `firmware/`) is the HLD's call. |
| AC-SIM-02a (partial) | Three scriptable switches with defined effects: **offline** (disconnect / stop all publishing → `offline` health, zone `offline`, gap record on return), **stale** (stay connected, stop telemetry → `stale-telemetry`, FR-TEL-04 alert, age badge), **implausible** (values outside declared range → `implausible-rejected` rows, device-health visibility, nothing on Home). Each switch is exercised in CI and in the 72 h soak. |

### 5.6 Platform & users (USR)

| ID | Acceptance criterion |
|---|---|
| AC-USR-01a | Console login = email + password + TOTP (enrolment on first login), session expiry, no anonymous route (security §2). |
| AC-USR-01b | Five roles `admin, grower, operator, technician, viewer` enforced **server-side** for every Phase 1 action per the security §2 matrix: user/role management and backup/restore = admin; device claim/assign/activate = technician/admin; alert ack/resolve = operator/grower/admin; zone active/idle set = operator/grower/admin; everything else read for viewer. Negative tests per role. |
| AC-USR-01c | Every operator action listed in security §5 that exists in Phase 1 (authn events, alert ack/resolve with reasons, backup/export/import, user/role changes, device provisioning/claim/activate, config edits) writes an audit row. |
| AC-USR-02a | Audit log is append-only: no update/delete path exists in the data-access layer and the DB policy refuses them (test attempts both). Audit rows are included in the export bundle. |
| AC-USR-03a | Settings shows service health, broker connected/disconnected, ingest lag, DB/storage growth; broker-disconnected and backup-failed are alertable. |
| AC-USR-04a | = §6.4 (org-scoping suite). |

### 5.7 Backup / export / restore (BKP)

| ID | Acceptance criterion |
|---|---|
| AC-BKP-01a | One-click export from Settings (IA J7) produces a single self-describing bundle: manifest with schema version, content types **and counts**, per-file checksums, manifest checksum. Entity types that do not exist yet (recipes, batches, inventory, orders) are declared absent in the manifest, not silently omitted. |
| AC-BKP-01b | Bundle contains: org/facility/room/rack/tier/zone, devices + profiles, users/roles, alert rules + history, audit log, telemetry rollups, and a raw-telemetry window whose size is declared in the manifest (default window is an HLD/LLD decision — **[unsourced]**). Export is audit-logged. |
| AC-BKP-02a | Import validates schema version and every checksum before writing anything; a tampered byte → refusal naming the file; an incompatible schema version → refusal with reason; operator chooses conflict policy (`import as new` vs `replace`) with warnings; import is audit-logged. |
| AC-BKP-03a | Scheduled backup runs at least daily by default (consistent with NFR-DAT-04's ≤24 h RPO recommendation), with retention pruning; a simulated failure (target unwritable) raises an alert; restore drill is documented in an operations runbook and **is** the §6.5 e2e. Off-host/offsite target is OQ-9 — not a Phase 1 gate. |

### 5.8 NFRs and ADR-0005

| ID | Acceptance criterion |
|---|---|
| AC-REL-01a | Stopping the backend container does not stop the virtual room controller or virtual racks (they keep connecting/publishing to the local broker); the broker is on the facility host, not a cloud endpoint. No Phase 1 service requires an internet connection to run (compose starts and passes e2e with outbound blocked). |
| AC-REL-02a | After a backend stop/start of duration D during telemetry flow, every device has a gap record covering D with cause `backend-down`; no `seq` discontinuity is left unaccounted (§6.1). Edge buffering/backfill is **not** required (3.4-e). |
| AC-PER-01a | With ≥50 zones publishing at the LLD-confirmed cadence (30–60 s class) plus scripted event bursts (all switches toggling, boot events), every published `seq` is stored or gap-recorded; ingest lag does not grow monotonically over the run. |
| AC-PER-02a | Home loads with facility state in <2 s and a tier drill-down in <3 s on the facility-host LAN against the 11-rack fixture with 72 h of history; measured at **p95 [proposed convention — qa to confirm]**, p50 also reported. |
| AC-ADR-0005a | = §6.4. Additionally the topic taxonomy carries `<org>/<site>` segments and broker ACLs are per device (security §3): a device credential cannot publish to another device's topics (tested against Mosquitto). |

### 5.9 Deliverable-level ACs (the five concrete deliverables)

| ID | Acceptance criterion |
|---|---|
| AC-DEL-01 | `docker-compose.prod.yml` runs Mosquitto (TLS :8883, per-device creds + ACLs), PostgreSQL 16 + TimescaleDB, the Go backend (authz, registry, ingest, alerts, backup modules) with secrets from docker secrets/env, nothing in repo. |
| AC-DEL-02 | Console v0 = Vue 3 + TS + Vite (ADR-0009), served as a static bundle by the Go backend, generated OpenAPI client only (ADR-0008 rule 1), five screens: Login, Home, Grow, Devices (incl. commissioning), Alerts, Settings (users, backup/export, self-observability). PWA caches shell only. Low-fi wireframes for these screens exist in `docs/ux/wireframes/` before LLD (IA §6). |
| AC-DEL-03 | Firmware v0: an ESP-IDF rack-controller skeleton that **builds in CI** with its own manifest (ADR-0008 rule 4) and implements capability advert, telemetry publish, and identify ack semantics; plus virtual device profiles. The transport the skeleton speaks *in Phase 1* is an HLD decision — see §9 concern 1. |
| AC-DEL-04 | Sim harness v0 per §5.5. |
| AC-DEL-05 | GitHub Actions deploy step wired by build-deploy-engineer to the facility host or dev box; Go, Node 20, and firmware CI jobs independently runnable; backup job scheduled; restore drill runbook committed. MQTT payload schema v1 is documented, versioned, and a hardware-side review is requested through the trophic-contracts CHANGELOG process (instrument-tags "hardware must review"). |

## 6. Definition of done for the six locked validation items

### 6.1 72 h virtual-facility soak

**Fixture:** 11 virtual racks × 4 tiers (44 zones) + virtual room controller, each device
publishing at its configured cadence; wall-clock 72 h (not accelerated — the property under
test is ingest/storage durability and rollup behaviour). Runs as a soak job on the deploy
host or dev box, producing a report artifact in `docs/pipeline/phase-1-foundation/`; CI runs
a short-form (minutes) version of the same script.

**Gap record (definition):** `(org, device, seq_from, seq_to | ts window, cause ∈
{device-offline, backend-down, broker-down, unknown}, detected_at)`. A gap is
**unaccounted** if a `seq` discontinuity in stored telemetry has no overlapping gap record.

**Done when, in the report:**
1. For every device, stored `seq` ∪ gap records covers the device's full published range;
   **unaccounted gaps = 0**. Gap records with cause `unknown` are permitted but each is
   explained in the report.
2. During the soak, scripted: ≥1 `offline`, ≥1 `stale`, ≥1 `implausible` on ≥1 device
   each, and ≥1 backend stop/start — each producing exactly the expected artefacts
   (gap record / stale alert raised then resolved / flagged rows, none on Home).
3. 1 m and 1 h aggregates complete for the whole window; last-known-state ages correct at
   spot checks; storage growth and ingest lag reported, lag non-monotonic.
4. No SAFETY alert raised by the soak fixture (it should not be able to — if one is, the
   rule or the fixture is wrong).

### 6.2 Commissioning round-trip

Done when a fresh virtual device, in one recorded e2e run: boots → appears `unclaimed`
with its profile → a guided-test request against it is refused → a `technician` user
claims it → displayed capabilities equal advertised (including one deliberately unknown
class shown as unrecognised) → assigned to rack N tier T → identify command round-trips
with `SUCCESS` ack and latency visible → activated → device shows `online` on Devices and
its zone on Grow → audit log contains one row per step with actor. An `operator` user
attempting the same flow is refused at claim.

### 6.3 Home <5 s

Two distinct measures, both sourced, both required:
- **Task measure (lock):** from authenticated navigation to Home, within 5 s wall-clock
  the screen shows all four Phase 1 inputs (alerts by severity with SAFETY pinned top;
  zones by status; device issues by health state; broker connectivity + last-sync/ingest
  lag) and every telemetry-derived number carries a source-age annotation; any value older
  than its class max-age carries a stale badge and is never rendered as current (IA
  principle 4, ADR-0009 caching boundary). Each tile links one tap deep.
- **Load measure (NFR-PER-02):** Home <2 s, tier drill-down <3 s, per AC-PER-02a.
- With a sim-driven SAFETY condition active, the alert is dominant on all five screens
  (AC-ALR-03a).
- Batches and desired-vs-actual tiles are absent (3.4-b).

### 6.4 Org-scoping suite (ADR-0005)

Done when: (1) the test database is seeded with two orgs; (2) **every** repository/data-
access method is invoked under org A and asserted to return or mutate zero org-B rows;
(3) a coverage gate enumerates repository methods and fails the build if any method lacks
such a test (mechanism — reflection, registry, or lint — is LLD's); (4) no API endpoint
accepts a client-supplied org id (org is resolved from the session); (5) MQTT topics carry
org/site segments and per-device ACLs hold (AC-ADR-0005a); (6) audit and telemetry tables
carry `org_id` like every core table.

### 6.5 Export / restore

Done when, after the soak: one-click export → bundle validates (manifest, checksums,
counts) → imported into a **clean** compose stack (fresh volumes) → import verifies
checksums and schema version, operator selects `import as new` → entity counts equal the
manifest; audit log restored intact and still append-only; rollups restored; virtual
devices reconnecting to the restored stack are recognised (same identities, same ACLs) and
telemetry resumes with a gap record covering the restore window. Negative paths: tampered
byte refused naming the file; incompatible schema refused with reason. Both export and
import appear in the audit log. The runbook used is the committed restore drill.

### 6.6 Security pass

Done when `05-security-review.md` records **no open BLOCK finding** on: authn (password +
TOTP, session expiry, no anonymous routes); server-side RBAC per AC-USR-01b; audit
append-only (AC-USR-02a); transport (MQTT TLS :8883, per-device credentials + topic ACLs,
device credential scheme — mTLS or PSK-with-justification — recorded in a new ADR per
security §2); secrets absent from repo (scanner in CI); backups access-controlled /
encrypted at rest and export audit-logged (security §1); console served with standard web
hardening, PWA caching shell only, no browser MQTT (ADR-0009); **no actuation path**
(AC-DEV-03e); SAFETY alert semantics (AC-ALR-03a); plausibility gates never drop
(AC-TEL-02a); every seeded threshold verbatim from `safety-rules.json` with the CO₂ mirror
labelled (AC-ALR-06a). Command topics never rely on QoS 0 (ADR-0004 consequence).

## 7. What "done" for Phase 1 as a whole means

All of §6.1–6.6 pass, every §5 AC passes or is explicitly waived by the owner in
`11-e2e-report.md`, the five deliverables of §5.9 exist on `main`, and the following are
recorded for the next release rather than fixed here: FR-BKP-04, NFR-UX-02, FR-SIM-02
remainder, FR-ALR-05, FR-DEV-05/06/07, edge buffer/backfill, the G1 EC-gate correction
reconciliation into `safety-rules.json` (contracts CHANGELOG 0.2.0 — maintainer loop), and
the RK-A-ROOM Rev 3 → Rev 4 citation update in device-control-model.

## 8. Roadmap and portfolio fit

- **Phase E three-year roadmap, Year 1:** "launch Phase 1 products 1.1–1.4 and 1.6";
  **Product 1.6 (CEA Environmental Monitoring Node, T/RH/CO₂/light, app dashboard)** is
  TRL 6-7 with the explicit rationale "the cloud dashboard/alerting software layer needs
  building and field validation". Phase 1 *is* that software layer, validated on the Ooty
  facility as Phase E prescribes ("using the Ooty facility itself as both a demonstrator
  and a genuine data-generating R&D instrument"). **Fits as planned.**
- **Product 2.1 (Multi-zone climate controller, Year 2, TRL 4-5)** is the roadmap's
  flagship; its "local-first control architecture with optional cloud dashboard" is
  exactly ADR-0001/0004. Phase 1 lays the registry/ingest/alert spine 2.1 will report
  into; nothing here pre-empts or forecloses it. No "ahead of schedule" element.
- **Framing nuance (not a deviation):** Phase C/E describe 1.6's software as a "cloud
  dashboard" with "recurring SaaS-style margin". The accepted architecture delivers an
  on-facility platform with the cloud layer optional and never in a control path
  (ADR-0001), single-tenant now and tenant-ready (ADR-0005). The SaaS angle is a
  productization decision deferred by ADR-0005's own trigger ("when the first external
  customer is real"). This was accepted at the ADR level; Phase 1 simply inherits it.
- **Portfolio items untouched:** Product 1.3 (NFT trays) is pending re-evaluation against
  flood-and-drain (owner 2026-09-07) — no Phase 1 bearing. Product 2.3 (saffron chamber)
  and the Phase E saffron moat weighting are orphaned by OQ-1 — no Phase 1 bearing.
- **Never-outsource IP:** Phase E §4.3 names firmware as in-house IP; firmware v0 stays in
  this repo under ADR-0008. Consistent.
- No market size, cost target, or calendar timeline is stated in the sources for this
  release and none is needed to scope it; the only durations used (72 h soak, <2/<3/<5 s,
  30–60 s cadence, ≤24 h RPO) are quoted from the lock, NFR-PER, NFR-DAT-04, or
  device-control-model.

## 9. Concerns and open items for the HLD (ranked; 1 is the one that could change the architecture)

1. **Firmware v0 rack-controller skeleton — what does it talk to in Phase 1?** Rack
   controllers are Modbus RTU nodes, not MQTT clients (ADR-0004 point 1: the room controller
   is the only path), and the Modbus register maps are an unpublished hardware deliverable
   (contracts README "What is missing"). So the ESP32 skeleton cannot implement the real
   bus contract, and letting it publish MQTT directly "for dev" would create exactly the
   second client type ADR-0004 forbids if it persisted. The HLD must decide the Phase 1
   transport for the skeleton (candidate: the skeleton exercises the capability-advert /
   telemetry / ack *payload semantics* and builds in CI; the virtual room controller is
   the only MQTT-facing thing; the bus contract lands in R2 with the register maps). This
   is the one item that shapes the firmware/sim boundary and ADR-0008 T1/T6 discussions.
2. **NFR-REL-02 "architecture shape"** — this note reads it as gap detection + recording
   in, edge buffer/backfill out (3.4-e). If the HLD disagrees, the 72 h soak definition
   changes.
3. **No desired state in Phase 1** cascades into: alert rules compare to rule thresholds
   (3.4-a); Home omits two IA tiles (3.4-b); zone active/idle is operator-set (3.4-c);
   DLI is an estimate from virtual light state (3.4-g). Each is a deliberate reading, not
   a gap — the HLD should carry them so R2 knows what it is upgrading.
4. **Two small ADRs are implied by Phase 1 code:** (a) the identify-class command in the
   capability vocabulary (ADR-0002 registry change, 3.4-d); (b) the device credential
   scheme, mTLS vs per-device PSK, which security §2 says must be "documented in an ADR
   when firmware work starts". Both start at ADR-0010 (0007 reserved for OQ-3).
5. **Process fit:** IA §6 gates hi-fi design on IA review + R1 scope freeze; the freeze
   exists, the review does not yet, and `ux-product-designer` is not wired into `/ship`
   (open-questions next action #6). The five screens need the low-fi wireframe pass before
   LLD (AC-DEL-02). Not architectural, but it will block step 6 if not scheduled.
6. **Unsourced values this PRD deliberately did not invent** — for LLD/qa to fix, not to
   guess: per-class staleness max-age (device-control-model §6 gives examples only),
   sensor cadence per class (NFR-PER-01 [A]), plausibility ranges per sensor (contracts:
   "sensor ranges, accuracy classes... not consolidated"), p95 vs other percentile for
   NFR-PER-02, raw-telemetry window in the export bundle, backup severity tier, VPD formula
   choice (botany review), offsite backup target (OQ-9).
7. **OQ-3 is not touched.** Nothing above depends on the room controller platform; the
   virtual room controller v0 is an MQTT client with no local TSDB/UI, and this note makes
   no recommendation on A/B/C/D.

## 10. Sources

`docs/roadmap/release-plan.md` (locked Phase 1) · `docs/requirements/requirements.md` ·
`docs/product/vision.md` · `docs/ux/information-architecture.md` ·
`docs/iot/device-control-model.md` · `docs/architecture/system-architecture.md` ·
`docs/data/data-architecture.md` · `docs/security/security-architecture.md` ·
`docs/decisions/open-questions-risks-next-actions.md` · `docs/adr/0001, 0002, 0004, 0005,
0006, 0008, 0009` · `docs/pipeline/phase-1-foundation/01-classification.md` ·
`docs/pipeline/oq3-room-controller-platform/00-decision-brief.md` ·
`.github/agentic-rules/safety-rules.json` · `knowledge/cea/README.md`,
`Phase_C_Product_Portfolio_Manufacturing_Feasibility.md`,
`Phase_E_Roadmap_TRL_CostMatrix_References.md` · `trophic-contracts` v0.2.0: `README.md`,
`CHANGELOG.md`, `capabilities/absent-by-design.md`, `sensors/instrument-tags.md`,
`conventions/units.md`.
