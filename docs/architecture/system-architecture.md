# System architecture

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** initial system architecture, reliability/safety architecture, and
simulation strategy of the foundation analysis. Sits on ADR-0001…0005. The physical
control allocation below is **the one already documented in the Rack A engineering set**
(RK-A-SYS/MFG/WRS/ROOM Rev 2–3) — the platform is built to slot into it, not to reinvent it.

## 1. Edge-first hybrid (ADR-0001) — three layers, documented

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ LAYER 4 — HARDWIRED SAFETY (no software, no network)                          │
│   E-stop circuit · NC fill / NO drain solenoid polarity · LSH-04 header switch│
│   leak puck→controller hard-wire · MV-01 spring-return closed · floor flood   │
│   sensor→MV-01 hard-wire · mechanical make-up float · overflows w/ air gaps   │
│   AUTHORITY: cannot be overridden by any software layer. Ever.                 │
├──────────────────────────────────────────────────────────────────────────────┤
│ LAYER 3 — EDGE CONTROL (facility-local; survives total internet/backend loss)  │
│   11× RACK CONTROLLERS (ESP32+8ch relay+RS485): tier fill/dwell/drain,        │
│     fan PWM, leak & header interlocks, last-known-good schedule               │
│   ROOM CONTROLLER (bus master, UPS-fed): drain token, 8-gate reuse, dosing,   │
│     HVAC & CO2 setpoints, alarms, local time-series DB, local UI               │
│   + Terrace I/O node · Water-skid node · Room I/O node (14 bus nodes,         │
│     Modbus RTU / RS485 multi-drop)                                            │
│   AUTHORITY: all actuation decisions. Cloud proposes; this layer validates.   │
├──────────────────────────────────────────────────────────────────────────────┤
│ LAYER 2 — FACILITY PLATFORM (single host, docker-compose; the "prod stack")   │
│   Mosquitto broker (TLS 8883) · backend (registry, auth, recipes, batches,     │
│     inventory, orders, analytics) · PostgreSQL+TimescaleDB · operator console │
│     · B2B portal (R5)                                                         │
│   ROLE: configuration authoring, desired-state *proposals*, history, R&D,     │
│     dashboards, alerts aggregation, export/backup                            │
├──────────────────────────────────────────────────────────────────────────────┤
│ LAYER 1 — REMOTE/CLOUD (optional; "remote view and history" only)             │
│   offsite backup sync · remote access path for P4 (Coimbatore) · later:       │
│     B2B portal publishing, multi-facility views                               │
│   ROLE: never in a control or safety path. Removing it changes nothing the     │
│     room depends on (documented: "nothing in the growing process depends on it")│
└──────────────────────────────────────────────────────────────────────────────┘
```

Deployment reality: today one host in the facility runs layers 2–3's broker-facing
services; the console and portal may be served from there (LAN) with remote access via
VPN/relay (R5 decision). The cloud layer stays optional per the engineering docs; the
compose stack is where it would live when productization needs it.

## 2. Service architecture (facility platform, Release 1 shape)

```
backend (Go, single modular service — boundaries as modules, not microservices)
         [ADR-0006, 2026-09-08 — supersedes the scaffold's inherited Node.js assumption;
          module boundaries below are unchanged by the language decision]
  ├─ authz          (users, roles, sessions, org scoping)
  ├─ registry       (facility/room/rack/tier/zone, devices, capability profiles)
  ├─ config         (desired-state authoring, recipe application, overrides)
  ├─ ingest         (MQTT client: telemetry, reported state, acks, events)
  ├─ alerts         (rule evaluation state machines, notification dispatch)
  ├─ production     (batches, harvests)        (R3)
  ├─ inventory      (lots, forecasts, orders)  (R4–5)
  ├─ analytics      (R&D query layer)          (R6)
  └─ backup         (export/import bundles, scheduled jobs)
frontend  (operator console, responsive web)     portal (R5, separate app)
firmware  (rack controller · room controller · I/O node profiles · virtual devices)
sim       (device simulators implementing the capability contract, FR-SIM-01/02)
```

Module boundaries are package-level (separate modules, shared DB, one process) — chosen
deliberately: one facility, one host, one deployable; modules give us the seams to
extract services later if scale demands (working rules 8, 10). Frontend and backend talk
over a versioned HTTP API (docs/api/ when written) + live updates over MQTT-over-WS or
SSE for dashboards (LLD decision).

## 3. Data flows

**Command/config flow (console → hardware):** console → backend API (authz + envelope
validation) → MQTT `cmd/config` → room controller (capability + safety-envelope +
interlock validation) → RS485 → rack controller/IO node → actuator; acks reverse the
path; every hop logged; desired-state retained on broker + persisted.

**Telemetry flow (hardware → everything):** sensors → rack/skid/room nodes →
room controller (local TSDB — the room's own black box) → MQTT → ingest service →
TimescaleDB (+ quality gates, alert state machines, rollups) → console/portal.

**Degraded-mode flows:** internet loss — layers 3–4 unaffected; host loss — room
controller keeps local logging/history (its own TSDB); backend restores by pulling the
gap from the room controller's local store (sequence-number-verified), explicitly
gap-marked (NFR-REL-02). Console loss — hardware runs; local UI on the room controller
remains (documented feature).

## 4. Time-series & relational strategy

Per [data-architecture.md](../data/data-architecture.md) (ADR-0004): PostgreSQL+Timescale
single store; hypertables for telemetry/desired-state; rollups serve dashboards; raw
retention tunable with export-before-rollup.

## 5. Identity, auth, security overlay

Per [security-architecture.md](../security/security-architecture.md): per-device broker
credentials + topic ACLs; user RBAC; audit append-only; org-scoping everywhere; E-stop
and interlocks off-network. OTA: signed, staged, rollback, window-restricted
(no updates mid-flood-window / active interlock).

## 6. Reliability & safety architecture (the brief's failure-mode questions)

| Scenario (from brief) | What happens (by design) |
|---|---|
| Internet fails | Nothing changes in the room (layers 3–4 autonomous); console (LAN) still live; remote access degrades; alerts queue; sync on restore with gap marks |
| Backend/host fails | Room controller continues all control + local logging + local UI; racks keep schedules; telemetry buffered in room controller; backfill on restore |
| Controller loses connection (bus drop) | Rack controllers run last-known-good tier schedules + local interlocks ("an alarm, not a stoppage"); fans/valves degrade per-class policy |
| Sensor impossible values | Quality gates flag (suspect/implausible), never dropped; cross-checks where present; device-health workflow; safety decisions never rest on one sensor |
| Pump fails | Duty/standby auto-changeover (P-02 A/B); 4-min run limit catches stuck floats; G4 filter-ΔP inhibit; maintenance alert with context |
| Water level insufficient | G8 holds recovery batches (TK-01 < 90%); make-up float is mechanical; MV-01 inhibited ≥ 28 °C solution temp; operator sees water inventory + alerts |
| Temperature exceeds safety limits | Room envelope alarms; HVAC/dehumidifier actuation per envelope; sourced limits only (`safety-rules.json`); SAFETY alerts audible/visible, cannot be dismissed without reason |
| Power restoration | Everything de-energizes to documented fail-safe states (drains open, fills closed, MV-01 closed, FV-01 to waste); controllers boot, self-check, resume schedules; no one-shot command replay; power events logged |
| Firmware update | Dual-partition, signed, staged (one rack first), rollback on failed health check; blocked during flood windows/active interlocks |
| Manual override | Time-boxed, bannered, audited, auto-reverts (device-control-model §8) |
| E-stop engaged | Hardwired scope drops (fills, MV-01, pumps; drains open); platform reflects + refuses conflicting commands; one-click incident package |

**Design law** (from safety-rules.json + engineering docs, restated as architecture):
no software layer may create an operating state that requires connectivity, the backend,
or any single sensor to *prevent* an unsafe physical condition. The cloud owns "remote
view and history and nothing the room depends on" — this sentence is an acceptance test
for every release.

## 7. Simulation strategy (FR-SIM-01..03)

- **Virtual devices** implement the capability contract (device-control-model §10);
  the entire platform stack runs against a simulated 11-rack facility from day one —
  software development is not gated on hardware availability.
- **Virtual room controller**: simulates the documented bus + gate logic (drain token,
  8-gate reuse) so orchestration bugs are found in CI, not in the room.
- **Fault injection matrix**: every row of §6 (offline, stale, impossible, command-fail,
  reboot, power-loss restore, e-stop) is a CI scenario; a reliability claim not
  reproduced in simulation is not a claim.
- **Environment dynamics (thermal/VPD/humidity simulation)**: deliberately deferred
  (FR-SIM-03) — physics fidelity is its own R&D problem; the *control* behaviors are
  what the platform must prove first.
- **HIL**: `automation-engineer`'s hardware-in-the-loop pass (pipeline step) runs the
  same tests against real ESP32 IO before firmware/hardware changes ship.

## 8. Multi-tenancy & productization path

[ADR-0005](../adr/0005-single-tenant-now-tenant-ready-schema.md): single-tenant now;
org-scoped schema; productization = deployment + auth + billing, not migration. Aquarium
product (P7) = separate console app reusing registry/ingest/config/alerts modules and the
capability vocabulary (ADR-0002); zero CEA UI sharing.

## 9. Deployment architecture

- Phase 1 (now): single facility host, `docker-compose.prod.yml` (services activate as
  releases land); GitHub Actions build→test→deploy on push to `main` (existing
  workflow, deploy step currently a placeholder — wired to the real host by
  `build-deploy-engineer` in R1).
- Secrets: docker secrets / env from restricted source; nothing in repo.
- Backups: scheduled on-host + offsite copy (FR-BKP-03), restore drill documented in
  operations runbook.
- Explicitly deferred: k8s, multi-region, per-PR integration stack
  (`docker-compose.integration.yml` — Phase 2 CI/CD per docs/pipeline/README.md).