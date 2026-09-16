# 02 — Domain note: control / IoT plane (Phase 1 foundation)

**Pipeline:** `/ship` · **Slug:** `phase-1-foundation` · **Author:** iot-control-systems-engineer
**Date:** 2026-09-10 · **Inputs read:** `01-classification.md`; `release-plan.md` Phase 1 lock;
`device-control-model.md` §1, §4, §5, §6, §9, §10; `system-architecture.md` §2, §3, §7;
ADR-0001/0002/0004/0008; `requirements.md` FR-DEV/FR-TEL/FR-SIM/NFR-REL;
`data-architecture.md` §2–3; `security-architecture.md` §2–4; `safety-rules.json`
(`valve_and_actuator_fail_safe_states`, `electrical_fail_safes`); Rack A Specification
RK-A-E01 (control multi-drop, ESP32 rack controller); Phase A §2.5; Phase D BOM §4
(sensors); trophic-contracts v0.2.0 `sensors/instrument-tags.md`,
`capabilities/absent-by-design.md`, `conventions/units.md`; OQ-3 decision brief.

> **Scope guard.** Phase 1 has no actuation, no closed loop, no desired-state store. This
> note designs the *plumbing* those will later ride on — topics, envelope, state machines,
> command/ack, sequence/backfill, sim placement, fault switches — and nothing that moves a
> valve, a fan, or a light. Where a number below is not sourced it is labelled
> **[DESIGN PARAMETER — proposed default]** and collected in §9 for the HLD to ratify.

---

## 0. Findings that shape everything else

1. **Production rack controllers have no network.** RK-A-E01 fixes the rack controller as
   "ESP32-class with 8-channel relay and RS485 transceiver" on a Modbus RTU multi-drop
   whose master is the room controller; ADR-0004 rule 1: *"No service ever talks to a rack
   controller or I/O node directly — the room controller is the only path."* Consequences:
   the ESP32 skeleton must **not** grow an MQTT/TLS stack (§1.4); rack-controller messages
   need a **link framing** distinct from the MQTT envelope; and the **room controller is the
   clock and sequence authority** for everything behind it (§4). Firmware-side Modbus
   register map is "not published" (instrument-tags.md), so the Phase 1 rack-link framing
   is an explicit placeholder, replaced in R2.
2. **A device is a bus node, not a sensor.** Identity + capability profile + credential
   live on the rack controller, room controller, and I/O nodes (the "14 bus nodes"). The
   T/RH node (I2C), leak puck (dry contact), LSH-04, tachos are *capabilities with a tag and
   a scope* inside the rack controller's profile. This removes the `<tier?>` segment from
   the topic in v1 (tier is a scope attribute in the payload) and keeps ACLs per node.
3. **No telemetry-timing number exists in any source.** The only 60 s in the source set is
   the leak-interlock solenoid closure proof (QC hold point) — it is a valve number and
   must not be borrowed as a heartbeat or staleness value. `device-control-model` §6's
   "T/RH 2 min, CO2 5 min" is an *example* in a proposed doc, not a sourced threshold. All
   timing in this note is therefore a design parameter (§9).
4. **Phase 1 profiles advertise only what the device can actually do.** Under ADR-0002 a
   control appears when the capability is advertised; Phase 1 cannot actuate, so Phase 1
   profiles advertise **sensing classes + `control.compute` + `sys.identify` only**.
   Actuator classes (`light.dim`, `airflow.speed`, `irrigation.flood`, `valve.control`)
   are added to the adverts in R2 when firmware can honour them. This is what makes the
   REJECT path (§3.4) a real test rather than a stub.
5. **Fail-safe preservation applies to the skeleton even without actuation.** The ESP32
   drives an 8-channel relay board whose channels switch the 24 V fill (NC) and drain (NO)
   solenoids. `safety-rules.json` de-energized states are CLOSED (fill) / OPEN (drain), set
   by solenoid polarity — firmware's only job is to never energize a relay. Requirement on
   `firmware-engineer`: relay GPIOs are driven to the *de-energized* level as the first
   action of boot, held there through reset/flash, and **the drive polarity of the actual
   relay module (many 8-ch boards are active-LOW) is verified on the bench** so that a
   floating pin during boot cannot chatter a relay. `electronics-hardware-specialist` to
   confirm. This is Phase 1 bring-up scope because the skeleton is the first code on that
   board.

---

## 1. Phase 1 MQTT topic set and payload schema v1

### 1.1 Topic grammar (v1)

Concretises `device-control-model` §4 / ADR-0004 rule 3 under the ADR-0004 root
`trophic/<org>/<site>/…`. Two refinements the HLD should fold back into §4:
`<room>` becomes an explicit segment (the registry hierarchy is facility → room → rack →
tier, and the room controller's ACL must be a topic *prefix*); `<tier?>` is dropped from
the topic and carried as `scope` in the payload (finding 0.2).

```
trophic/<org>/<site>/<room>/<zone>/<device>/<leaf>

<org>     tenant slug (ADR-0005)                        e.g. trophic
<site>    facility slug                                 e.g. ooty-r1
<room>    room slug                                     e.g. room-1
<zone>    rack-<nn> | room                              e.g. rack-01, room
<device>  registry device_id of the bus node            e.g. rc-01 (rack ctl), rmc-01 (room ctl), io-terrace
<leaf>    capability            retained=true  QoS1   capability advert (§1.3)
          state/health          retained=true  QoS1   heartbeat + LWT (§1.6)
          state/reported        retained=true  QoS1   defined; unused in Phase 1 (no actuators advertised)
          state/desired         retained=true  QoS1   defined; unused in Phase 1 (desired-state store is R2)
          telemetry/<metric>    retained=false QoS1   samples (§1.4)
          cmd                   retained=false QoS1   backend → device (§1.5)
          ack                   retained=false QoS1   device → backend (§1.5)
          event                 retained=false QoS1   lifecycle: boot, claim, link_gap, buffer_overflow, schema_unsupported
```

Rules:
- **Topics route and authorise; payloads identify.** Ingest resolves `device_id` from the
  payload envelope and cross-checks it against the topic prefix; a mismatch is stored with
  `quality: suspect` and raises a security event (never silently trusted, never dropped).
- **Publisher ACLs** (security-architecture §3): the room controller principal may publish
  under `trophic/<org>/<site>/<room>/#` (it bridges every node in its room) and subscribe
  to `…/<room>/+/+/cmd`. The backend ingest principal subscribes `trophic/<org>/<site>/#`
  and publishes only `…/cmd` and `…/state/desired`. A Phase 1 bench-only direct-MQTT
  principal for an ESP32 (if ever created, see §1.7) is confined to its own
  `…/<zone>/<device>/#` and never gets a production credential.
- **QoS ≥ 1 everywhere; `cmd` and `ack` never QoS 0** (ADR-0004 consequence).
- Version lives in the payload (`v`), not the topic root, so a v2 payload can coexist on
  the same topics and an old consumer sees the version before parsing (ADR-0002 rule 3).
  A topic-root bump is reserved for an incompatible *taxonomy* change only.

### 1.2 Common envelope (all messages)

```json
{
  "v": 1,                                  // payload schema major version (integer)
  "msg": "telemetry",                      // capability | health | telemetry | cmd | ack | event | state
  "org": "trophic", "site": "ooty-r1", "room": "room-1",
  "zone": "rack-01",                       // rack-<nn> | room
  "device_id": "rc-01",
  "boot_id": "b7f3a1",                     // publisher-assigned, changes on every boot of the *publisher*
  "seq": 4711,                             // per (device_id, boot_id), monotonic from 1, assigned by the publisher (§4)
  "ts_source": "2026-09-10T09:14:02.130Z", // wall clock of the *publishing* node (room controller)
  "t_mono_ms": 812934,                     // originating device monotonic ms (rack-link nodes), optional
  "clock": "ntp",                          // ntp | rtc | unsynced   (of the publisher's wall clock)
  "sim": false,                            // FR-SIM-01 marker; immutable after claim (registry rejects flips)
  "backfill": false                        // true when replayed from the outbound buffer (§4)
}
```

Ingest adds `ts_ingest` (broker-receipt time at the backend). **Staleness and health use
`ts_ingest`; series time uses `ts_source` when `clock ∈ {ntp, rtc}`, else `ts_ingest` with
`time_basis: ingest`** (FR-TEL-01 preserves the source clock either way).

### 1.3 Capability advert — `…/capability` (retained)

Published on every boot and on any profile change; the retained copy is what the registry
re-reads after its own restart (discovery does not depend on catching a live event).

```json
{ "...envelope, msg: capability",
  "profile": {
    "schema_version": "1.0",                       // ADR-0002 profile schema, validated at provisioning
    "device_type": "rack-controller",              // descriptive only — never a behaviour switch (ADR-0002 rule 4)
    "role": "rack",                                // control.compute role: rack | room | io-node
    "hw_rev": "RK-A-E01 r2", "fw_version": "0.1.0",
    "sim": false,
    "link": "rack-link/0",                         // how this node reaches the MQTT publisher (§1.7); "mqtt" for the room controller
    "capabilities": [
      { "class": "sys.identify", "scope": "device", "params": { "max_duration_s": 60 } },
      { "class": "control.compute", "scope": "device" },
      { "class": "sense.temp_rh", "scope": "rack-01/tier-3", "tag": "TRH-01",
        "metrics": [ { "name": "air_temp_c", "unit": "degC", "range": [-40, 125], "resolution": 0.01, "accuracy": 0.3 },
                     { "name": "air_rh_pct", "unit": "pct",  "range": [0, 100],   "resolution": 0.01, "accuracy": 2.0 } ],
        "interval_s": 30 },
      { "class": "sense.leak",        "scope": "rack-01", "tag": "LD-01",  "metrics": [ { "name": "leak", "unit": "bool" } ], "interval_s": 30, "on_change": true },
      { "class": "sense.water_level", "scope": "rack-01", "tag": "LSH-04", "kind": "switch",
        "metrics": [ { "name": "header_high", "unit": "bool" } ], "interval_s": 30, "on_change": true }
    ],
    "commands": [ "sys.identify" ]
  } }
```

- **Plausibility ranges are discovered, not tabled.** `range` is the sensor's *physical*
  range from the advert (device-control-model §6: "values from the sensor's documented
  range, not crop targets"). The backend keeps a per-class fallback table used only when a
  profile omits `range`, and records that it fell back. Accuracy figures above are the
  Phase D BOM figures for the SHT-class T/RH sensor (±0.3 °C / ±2 % RH); the T/RH range
  is the sensor-class range and must be confirmed by `electronics-hardware-specialist`
  against the chosen part — until then the virtual profile carries it and the real
  skeleton reports what its driver knows.
- `unit` strings must be from the allowed list derived from `conventions/units.md`
  (`degC`, `pct`, `ppm`, `umol_mol`, `kPa`, `mS_cm`, `L_min`, `m3_h`, `m_s`, `rpm`, `bool`);
  unknown unit → sample stored `suspect` + event, never dropped.
- **Unknown class / newer `schema_version` / newer `v`** → stored raw, ignored for control
  purposes, `event{type: schema_unsupported}`; device remains visible with a badge
  (ADR-0002 rule 3). Never a crash, never a silent config.
- **Profile change on a claimed device** → registry diffs and moves the device from
  `active` back to `claimed` ("re-verify capabilities"), keeps telemetry flowing, records
  the diff. A profile that adds a capability never auto-enables it.
- The virtual profiles **must not** advertise anything in `absent-by-design.md`: no
  per-tray level, no per-tier CO2, no fixed PPFD, no per-tier climate control. Rack T/RH is
  one `sense.temp_rh` at scope `rack-NN/tier-3`. VPD is computed by the backend
  (FR-TEL-03), never advertised as a metric.

**Phase 1 profile set** (what is actually advertised):

| Device | Profile contents | Notes |
|---|---|---|
| Virtual rack controller `rc-NN` (× 11 in the acceptance run; 1 in the base harness, 4 tiers from the profile) | `sys.identify`, `control.compute{role: rack}`, `sense.temp_rh @ tier-3`, `sense.leak`, `sense.water_level{LSH-04}` | Tier count comes from the profile (`tiers: 4`), never a constant in sim or backend. Tachos join in R2 with `airflow.speed`. |
| Virtual room controller `rmc-01` | `sys.identify`, `control.compute{role: room}`, `sense.temp_rh @ room`, `sense.co2 @ room (control NDIR)`, `sense.co2 @ room {role: safety-mirror}` | CO2 NDIR is in the Phase D BOM (SCD41 class, 400–5000 ppm) and the room spec (two NDIRs). The safety NDIR sample is a **mirror** of an independent hardware alarm (safety-rules `co2_safety_alarm_ppm`); the profile carries `role: safety-mirror` so FR-ALR seed rules label it as such. |
| Real ESP32 skeleton `rc-bench-01` | `sys.identify`, `control.compute{role: rack}`, plus `sense.temp_rh` **only if** an SHT-class sensor is detected on I2C at boot, `sense.leak` only if the dry contact is wired | Advert is built from what the firmware *detects/configures*, not a compile-time table — that is the discovery principle in practice. |

### 1.4 Telemetry — `…/telemetry/<metric>`

One sample per message (simple to ACL, simple to gap-detect; batching is an R2 optimisation
if volume ever demands — at Phase 1 rates it does not, see §9).

```json
{ "...envelope, msg: telemetry",
  "metric": "air_temp_c", "value": 22.41, "unit": "degC",
  "scope": "rack-01/tier-3", "tag": "TRH-01",
  "quality": "good",                 // device-side verdict: good | suspect (backend may downgrade, never upgrade)
  "calibration_id": null             // FR-DEV-07 is R2; field present, nullable, from day one
}
```

- Switch-type metrics (`leak`, `header_high`) publish **on change and every `interval_s`**
  so staleness is uniform across metric kinds.
- `quality` is the *device's* verdict; the backend applies §6 gates and may only downgrade
  (`good → suspect → implausible-rejected`). Rejected samples are **stored** with the flag
  (FR-TEL-02) — the harness's `implausible` switch (§6) proves the row exists.

### 1.5 Command and ack — `…/cmd`, `…/ack`

```json
// cmd (backend → device; Phase 1: only sys.identify)
{ "...envelope, msg: cmd, device_id: <target>",
  "cmd_id": "01J8ZK3V9R6Q2M0XQ7F1T4S8YC",     // ULID/UUIDv7 — the idempotency key
  "capability": "sys.identify", "action": "identify",
  "params": { "duration_s": 10 },
  "issued_at": "…", "expires_at": "…",       // TTL; expired commands are never applied late
  "actor": { "kind": "user", "id": "u_…" },  // security-architecture §4 (identity travels with the command)
  "cmd_seq": 88                              // backend's monotonic per-device command sequence
}

// ack (device/edge → backend)
{ "...envelope, msg: ack",
  "cmd_id": "01J8ZK3V9R6Q2M0XQ7F1T4S8YC",
  "result": "SUCCESS",                        // SUCCESS | REJECTED | TIMEOUT | EXPIRED
  "reason_code": null,                        // enum, see §3.4
  "reason": null,                             // human-readable, optional
  "duplicate": false,                         // true when this ack is a replay for an already-seen cmd_id
  "latency_ms": 143,                          // edge-measured, for FR-DEV-05 later
  "state_after": { "identify": { "active": true, "until": "…" } }
}
```

### 1.6 Health / heartbeat — `…/state/health` (retained) + LWT

```json
{ "...envelope, msg: health",
  "status": "online",                        // online | degraded | offline
  "reason": null,                            // for offline: lwt | shutdown | bus_timeout ; for degraded: clock_unsynced | buffer_dropped | sensor_init_failed
  "uptime_s": 81234, "fw_version": "0.1.0",
  "link": { "kind": "rack-link/0", "errors": 0 },
  "buffer": { "depth": 0, "capacity": 43200, "dropped": 0 },   // outbound ring (§4); omitted for rack-link nodes in v0
  "heartbeat_s": 30
}
```

- The **room controller's MQTT session carries a Last Will** on its own health topic:
  `{status: offline, reason: lwt}`, retained. Offline detection of the edge therefore
  happens in the broker, not in the backend, and needs no cloud.
- For nodes behind it, the room controller publishes their health **on their behalf**
  (`bus_timeout` when the rack-link goes quiet). When the room controller itself goes
  offline, the backend derives `offline (inherited)` for every node under it — no
  per-node LWT is possible for nodes that are not MQTT clients.
- `degraded` is self-reported and does not change the connectivity axis (§2).

### 1.7 The rack-link (ESP32 skeleton ↔ room controller) — Phase 1 placeholder

Decision proposed for the HLD: the ESP32 skeleton emits **the same v1 payload objects
(§1.2–1.6, minus `org/site/room`, `ts_source`, `clock`) as newline-delimited JSON over
UART** (USB-serial on the bench; the RS485 transceiver when fitted), and the (virtual)
room controller bridges them to MQTT, stamping `ts_source`, `clock`, `seq`, `boot_id`
(§4). Called `rack-link/0`; explicitly a placeholder for the Modbus register map that R2
must publish (instrument-tags.md "still missing"). Rationale:

- Keeps the ADR-0004 choke point real from the first commit: the rack controller never
  has broker credentials, an MQTT stack, or a TLS stack it will not ship with.
- Keeps the ESP32 clock-free: production rack controllers have no NTP path (no network),
  so wall-clock authority has to be the room controller anyway.
- The virtual room controller's bridge is the same code path for virtual devices
  (in-process channel) and the real skeleton (serial adapter) — so the acceptance criterion
  "≥1 real rack controller if hardware ready" is the same test with one adapter swapped.

Direct-MQTT-from-ESP32 is **not recommended**. If `firmware-engineer` needs it for
bring-up, it is a compile-time bench flag, a distinct broker principal confined to its own
device prefix, and it is deleted before R2 — it must never become the de facto path.

### 1.8 Output required: hardware-side review of schema v1

`instrument-tags.md` records the MQTT payload schema as "owned by the software side
(ADR-0004); hardware must review". The HLD must list as a Phase 1 **output**:

> **Schema v1 hardware review** — `docs/iot/schema/v1/` (JSON Schema per message type +
> golden examples) reviewed by the hardware program; result recorded as a
> trophic-contracts change (bump the "MQTT payload schema version" row from "must review"
> to "v1 reviewed <date>") and, in R2, the Modbus register map that carries the same
> fields over `rack-link`.

---

## 2. Device state machines (Phase 1)

Two orthogonal machines. **Lifecycle** is owned by the registry (operator actions);
**health** is derived from the broker/heartbeat/telemetry and applies from first sight.

### 2.1 Lifecycle (registry-owned, FR-DEV-01/03)

```
            first capability advert seen
 (none) ───────────────────────────────► UNCLAIMED
                                            │ operator "claim" (UC-04 step 1; identity/fingerprint recorded)
                                            ▼
                                         CLAIMED  ──(profile diff after activation)──┐
                                            │ operator verifies advertised profile   │
                                            ▼                                        │
                                       COMMISSIONED  (zone assigned: rack/tier)      │
                                            │ guided test = sys.identify round-trip   │
                                            │ with SUCCESS ack visible (§3)           │
                                            ▼                                        │
                                          ACTIVE ◄───────────────────────────────────┘ (drops to CLAIMED, keeps telemetry)
                                            │
                        operator retire ────┴──► RETIRED      fingerprint mismatch ───► QUARANTINED (security-arch §2)
```

- **UNCLAIMED devices are untrusted:** telemetry is stored (marked `unclaimed`, not joined
  to any zone) so the commissioning screen can show "seen N s ago"; **no command is ever
  published to an unclaimed device** (security-architecture §2). The edge also rejects
  commands to unclaimed nodes with `device_unclaimed` in case the backend rule is
  bypassed.
- Every transition is an audit row with actor + timestamp (FR-DEV-03 "every step
  recorded").
- `sim: true` is fixed at claim time from the advert; a later advert flipping it is a
  profile diff → CLAIMED + security event.

### 2.2 Health (derived, FR-DEV-04)

Two axes, surfaced as one badge:

```
Connectivity axis (heartbeat / LWT / bus):
   UNKNOWN ──first message──► ONLINE ◄──heartbeat resumes── OFFLINE
                                │  no heartbeat AND no telemetry for offline_after_s,
                                │  OR LWT fired, OR health{offline} received,
                                │  OR parent room controller OFFLINE (→ "offline, inherited")
                                ▼
                             OFFLINE

Telemetry axis (per advertised metric, using ts_ingest):
   FRESH ──age > max_age(metric)──► STALE ──new sample──► FRESH
   Device badge STALE-TELEMETRY = connectivity ONLINE ∧ any advertised metric STALE.

Self-reported: DEGRADED (health.status) overlays either axis; FAULT reserved for event{type: fault} (R2 use).
```

- Precedence for the single badge: `OFFLINE` > `FAULT` > `DEGRADED` > `STALE-TELEMETRY` >
  `ONLINE`.
- **`STALE` requires the device to be ONLINE.** A device that is offline is not "stale";
  its metrics show age but the badge is OFFLINE. This is what the `stale` fault switch
  (§6) is built to prove: heartbeats continue, samples stop.
- Staleness is per **advertised** metric; a metric the profile does not list cannot go
  stale. This is the discovery principle again — no "every rack must report tacho" rule
  hardcoded in the backend.
- Alerting: crossing into STALE-TELEMETRY raises an **OPERATIONAL** alert (FR-TEL-04);
  OFFLINE raises OPERATIONAL in Phase 1 (FR-ALR-05's richer device-failure rules are R2).
  Alerts carry the threshold that fired and cite it as *design parameter, §9 of this note /
  HLD* — not `safety-rules.json`, which has no such value (FR-ALR-06 honesty).
- Timing (all **[DESIGN PARAMETER — proposed default]**, see §9): heartbeat 30 s;
  `offline_after_s` = 3 missed heartbeats = 90 s; `max_age` per metric = max(3 ×
  `interval_s` from the advert, class floor) with class floors T/RH 120 s, CO2 300 s,
  switches 120 s (floors adopted from device-control-model §6's example, which is the
  current spec's own proposal and not a sourced threshold).
- Production note (not Phase 1 code): for real rack controllers the *authoritative* offline
  verdict is the room controller's bus timeout (Modbus timeouts are milliseconds); the
  backend's heartbeat window is a mirror with a coarser clock. The room controller's own
  offline is authoritative from the broker LWT. Nothing in the room waits on the backend's
  verdict.

---

## 3. Command / ack semantics — `sys.identify` (commissioning round-trip)

### 3.1 What identify is

A **no-op with an observable, side-effect-free indication**: blink the controller's
on-board status LED (or a log line / a virtual "identify active" flag) for `duration_s`.
**It must never touch the 0–10 V dimming pair, a relay channel, a fan PWM line, or any grow
LED.** `firmware-engineer` implements it against the status LED only; the virtual device
sets a visible flag the console can render. `max_duration_s` is advertised in the profile
(proposed 60 s) and enforced at the device — a request above it is `params_out_of_range`.

### 3.2 Pipeline (Phase 1 instantiation of device-control-model §5)

```
console ──► backend: authz (technician/admin), device ACTIVE or COMMISSIONED (guided-test step),
            capability "sys.identify" ∈ registry profile, params ≤ advertised max
        ──► publish cmd (QoS1) with cmd_id, expires_at = issued_at + ttl_s
(virtual) room controller: re-validates against the *device's own* advert (not the registry copy),
            device claimed?, not expired?, cmd_id seen before?
   ├─ seen before → republish stored ack, duplicate: true      (idempotency)
   ├─ REJECT     → ack{REJECTED, reason_code}
   └─ ACCEPT     → forward over rack-link / in-process; wait ack_timeout_s
        ├─ device SUCCESS → ack{SUCCESS, state_after, latency_ms}
        ├─ device REJECT  → ack{REJECTED, reason_code} (device-side verdict wins)
        ├─ no reply       → retry (same cmd_id) up to max_retries, then ack{TIMEOUT}; device health → DEGRADED{reason: command_timeout}
        └─ device OFFLINE → park until expires_at; if it returns in time, deliver; else ack{EXPIRED} + event
backend: on ack → command record {issued, acked, result, latency}; if no ack by backend_wait_s → record "unacknowledged"
         (a backend-side state — the backend never synthesises an ack message on the ack topic)
```

### 3.3 Idempotency

- `cmd_id` is the idempotency key. The edge keeps a **dedup ring of the last N cmd_ids per
  device with their acks** (N proposed 32) and replays the stored ack for a duplicate — the
  device is not re-poked. Broker redelivery, operator double-tap and edge retry all
  collapse to one effect.
- `cmd_seq` (backend monotonic per device) lets the edge detect out-of-order or replayed
  commands from a stale backend instance; a `cmd_seq` lower than the last accepted one is
  `REJECTED{stale_sequence}` unless it is a duplicate `cmd_id`.
- Identify is naturally idempotent (re-applying = still blinking). The ring exists because
  R2 commands will not be, and the semantics must not change between releases.

### 3.4 REJECT path — the capability test

The acceptance test for ADR-0002's "capability absent/incompatible is a first-class
outcome": the console (or a test client) publishes `{capability: "light.dim", action: "set",
params: {level: 70}}` to a Phase 1 virtual rack controller. The virtual device's advert does
not list `light.dim` → `ack{REJECTED, reason_code: capability_not_advertised}`. The backend
records it, the console shows it with the reason, and a repeat-rejection counter feeds the
security-architecture §4 "repeated rejections from one source" rule (alert wiring is R2;
counter exists now).

Note the backend *also* refuses to publish it (fast-path validation, ADR-0001 rule 3), so
the test must publish past the backend to exercise the edge — the harness test client does
this; the console cannot.

`reason_code` enum v1 (closed set; additions bump the schema minor):

| code | Phase 1 use | meaning |
|---|---|---|
| `capability_not_advertised` | yes | class not in this device's profile |
| `action_not_supported` | yes | class advertised, action unknown |
| `params_out_of_range` | yes | outside advertised params (e.g. `duration_s` > `max_duration_s`) |
| `schema_version_unsupported` | yes | `v` newer than the device understands |
| `device_unclaimed` | yes | edge refuses commands to unclaimed nodes |
| `stale_sequence` | yes | `cmd_seq` regression, not a duplicate |
| `expired` | yes | `expires_at` passed before delivery |
| `busy` | yes | another command in flight on a single-slot capability (identify is single-slot per device) |
| `interlock_active`, `envelope_violation`, `e_stop`, `not_in_window` | **reserved (R2)** | defined now so the enum does not change when actuation arrives |

### 3.5 Timing — all [DESIGN PARAMETER — proposed default] (§9)

`ack_timeout_s` 5, `max_retries` 2 (3 attempts, ~15 s worst case), `ttl_s` 60 for
identify, `backend_wait_s` 30 (must exceed edge worst case), dedup ring 32. There is no
sourced round-trip figure; the identify command has no physical consequence, so these
are UX numbers and the HLD may tune them freely — but they become the *defaults* that R2
actuator commands inherit unless a class overrides them, so they should be recorded, not
buried.

---

## 4. Sequence numbers, buffering, and backfill (NFR-REL-02)

### 4.1 Who owns `seq`

`seq` is assigned by the **MQTT publisher** — the room controller — per `device_id` stream
it publishes, restarting at 1 under a new `boot_id` when the *publisher* reboots, and
persisted alongside its outbound buffer so a publisher restart does not silently reuse
numbers (a new `boot_id` makes the reset explicit). Rack-link nodes carry their own
`link_seq` (per their own boot) which the room controller preserves as `t_mono_ms` +
`link_seq` for diagnostics and uses to detect **bus-level** loss, which it reports as
`event{type: link_gap, device_id, from, to}`. Backend gap detection runs over the
publisher's `seq` because that is the sequence space the buffer can fill.

### 4.2 What Phase 1 must implement

| Item | Phase 1 | Why now |
|---|---|---|
| `seq`, `boot_id`, `backfill` in every envelope | **must** | Schema v1 is frozen by hardware review; retrofitting is a v2 |
| Ingest gap detection per (device, boot_id): missing `seq` → `telemetry_gaps` row `{device_id, boot_id, seq_from, seq_to, detected_at, status: open}` | **must** | "72 h with zero gaps unaccounted (gaps must be recorded gaps)" is the acceptance criterion |
| Room controller **outbound ring buffer**, bounded, persisted (file-backed in the virtual controller), replay in `seq` order on reconnect with `backfill: true`; on overflow, drop oldest and publish `event{type: buffer_overflow, seq_from, seq_to}` | **must** | Host/broker outage is the §6 row "Backend/host fails → telemetry buffered"; without it 72 h has unrecorded gaps |
| Broker-side persistence for the backend's subscription (persistent session, QoS1, `max_queued_messages` raised well above Mosquitto's small default and sized to the outage window) so a backend-only outage needs no device replay | **must (infra)** | Two different outage shapes; both must be reproduced by the harness (§6) |
| Gap row → `filled` when the replay covers it; `unfillable` when a `buffer_overflow` event covers the range | **must** | "Recorded gaps" means the row says *why* it stayed open |
| Backfilled samples: stored normally with `backfill: true`, `ingest_lag_s`; **do not** advance the live "last known state", **do not** trigger staleness recovery, **do not** evaluate alert rules (they are history). A live `telemetry_gaps` open row raises one OPERATIONAL "telemetry gap" alert per device | **must** | Prevents a 4 h replay from re-raising 4 h of stale alerts |
| **Pull backfill** — backend asks the room controller for a `seq` range from its local TSDB (`sys.backfill` command) | **R2** | Needs the real room controller's local store, which is OQ-3 / ADR-0007 territory. Phase 1 replay is *push-on-reconnect from the ring* only |
| Rack-link nodes buffering | **R2 (if at all)** | ESP32 v0 has no buffer; a `link_gap` in Phase 1 is `unfillable` by construction and recorded as such |

Ring capacity **[DESIGN PARAMETER — proposed default]**: 6 h at nominal rate for the virtual
facility (11 racks × ~3 metrics + 4 room metrics at 30 s ≈ 74 msg/min ≈ 27 k messages —
`capacity` is reported in health so the console can show "buffer 12 % full"). The real room
controller sizes to multi-day on disk (OQ-3 brief R11); not a Phase 1 number.

---

## 5. Sim harness placement (ADR-0008) and OQ-3 neutrality

### 5.1 Placement: top-level `sim/`

`system-architecture.md` §2 already lists `sim` as a top-level component next to
`firmware`. Proposed layout, each piece with its own manifest (ADR-0008 rule 4):

```
sim/                          own Go module (sim/go.mod) — Go for owner fluency (ADR-0006), NOT a backend package
  cmd/simctl/                 CLI: run a scenario, flip fault switches
  cmd/virtual-room/           the virtual room controller (MQTT client, bridge, buffer, health, cmd router)
  internal/vdevice/           virtual device runtime: loads a profile, emits samples, answers cmd
  internal/racklink/          rack-link/0 adapters: in-process channel; serial (JSON-lines) for the real ESP32
  internal/faults/            offline / stale / implausible switches (§6)
  profiles/                   capability profile JSON: rack-controller-v0.json, room-controller-v0.json, esp32-skeleton-v0.json (golden capture of the real advert)
  scenarios/                  YAML timelines for CI (e.g. 72h-soak.yaml, backend-outage-20m.yaml, reject-path.yaml)
docs/iot/schema/v1/           JSON Schema per msg type + golden example messages — the contract of record
```

Boundary rules applied:
- `sim/` imports nothing from `backend/`, `frontend/`, or `firmware/`; `backend/` imports
  nothing from `sim/`. They meet only at the broker and at `docs/iot/schema/v1/`, which
  plays the role OpenAPI plays between backend and frontend (ADR-0008 rule 1): **each
  subsystem validates against the schema files; none shares Go types by path.** CI runs a
  conformance test in `sim/`, `backend/`, and `firmware/` (host-side unit test parsing and
  emitting the golden messages).
- `firmware/` contains the ESP32 skeleton only (ESP-IDF, own manifest). The virtual
  devices are **not** a build target under `firmware/`: the release plan's "virtual device
  profiles" are data files, and `sim/profiles/esp32-skeleton-v0.json` is a *captured*
  advert, not a shared source. This keeps a later firmware/hardware extraction (ADR-0008's
  most likely split shape) a directory move.
- The `docker-compose.prod.yml` stack does not include `sim/`; a dev/CI compose overlay
  adds `virtual-room` as a service. (Phase 2's per-PR integration stack is still not
  built — this is an overlay, not that.)

### 5.2 Keeping the virtual room controller platform-agnostic

The virtual room controller is a **protocol- and behaviour-faithful stand-in**, not a
prototype of the real one. Constraints so it cannot pre-empt OQ-3:

1. It implements exactly the observable contract in §1–§4: topics, envelope, LWT, health
   on behalf of nodes, cmd validation against adverts, retry/timeout/park, outbound ring
   with replay, gap events. **Nothing else** — no local TSDB, no local UI, no drain token,
   no 8-gate logic, no Modbus (those are R2+ and, for the real controller, ADR-0007).
2. All platform touchpoints are behind small interfaces: `Clock`, `BufferStore` (memory or
   file), `RackLink` (in-process / serial), `Broker` (MQTT client). No filesystem layout,
   no daemon/init assumptions, no GPIO, no serial device path outside the adapter.
3. Its README states: *"This is a simulator. Whether any of it becomes the real room
   controller is decided by ADR-0007, not by this code existing."* If OQ-3 lands on Linux +
   Go, reuse is permitted as a separate decision; T1 of ADR-0008 fires then regardless.
4. If OQ-3 lands on option D (MCU bus master + Linux supervisor) the split lands inside
   `RackLink`, and nothing in the backend or schema changes — which is the property we
   want to have proven by then.

---

## 6. Fault switches — concrete behaviour and test oracles

The release plan locks fault switches (offline, stale, implausible) into Phase 1 although
`requirements.md` files them as FR-SIM-02 (S, R2). **FR-ID drift noted, not re-litigated**;
the HLD should either move the three switches into FR-SIM-01 v0 or annotate FR-SIM-02
"offline/stale/implausible delivered in R1". Command-failure, reboot, power-loss-restore
and E-stop switches remain R2.

**Control surface:** switches are flipped through `simctl` (local CLI / HTTP on the sim
process) or a scenario file — **never through the device `cmd` schema.** There is no
fault-injection command in v1; a real device must not be able to receive one.

| Switch | Variant | What the sim does | Observable oracle (what the test asserts) |
|---|---|---|---|
| `offline` | `device` (rack node behind the room controller) | Rack-link adapter stops delivering for `rc-NN`; after `link_timeout_s` the virtual room controller publishes retained `health{offline, bus_timeout}` for `rc-NN` and `event{link_gap}` when it returns | Backend health → OFFLINE within `offline_after_s`; console badge; a `sys.identify` issued while offline is parked and returns `EXPIRED` after `ttl_s` (or SUCCESS if the switch clears in time); `telemetry_gaps` row `unfillable` for the link gap |
| | `edge-ungraceful` (room controller) | TCP socket dropped without MQTT DISCONNECT; process keeps running and **buffering** | Broker fires LWT → retained `health{offline, lwt}` for `rmc-01`; every node under it → OFFLINE (inherited); on reconnect: replay in `seq` order with `backfill: true`, `telemetry_gaps` rows → `filled`; alert rules **not** re-evaluated on replayed samples |
| | `edge-graceful` | Publishes `health{offline, shutdown}` then DISCONNECTs; stops buffering | OFFLINE immediately with reason `shutdown`; gap on resume is `open` → later `unfillable` (nothing was buffered) with a `buffer_overflow`-class reason `not_buffered` |
| | `backend-only` (infra switch, not sim) | Backend container stopped; broker + sim keep running | Broker persistent session queues; on backend restart no `telemetry_gaps` row is created (proves broker sizing); if the queue limit is hit, the gap **is** detected — that is the point |
| `stale` | `metric` | One advertised metric stops publishing; heartbeat and other metrics continue | Metric age grows; device badge → STALE-TELEMETRY (not OFFLINE) after `max_age(metric)`; one OPERATIONAL alert citing the design-parameter threshold; recovers on first new sample |
| | `all` | All telemetry stops; heartbeat continues | Same, all metrics; device stays ONLINE on the connectivity axis |
| | `clock-frozen` | Samples continue with `ts_source` frozen | Ingest detects non-advancing `ts_source` vs advancing `ts_ingest` → samples `suspect`, `time_basis: ingest`, `event{clock_anomaly}`; **no** staleness (data is arriving) — proves staleness is on `ts_ingest` |
| `implausible` | `range` | Publishes a value outside the advertised `range` (RH 118 %, T −55 °C, CO2 12 000 ppm) | Row stored with `quality: implausible-rejected`; excluded from last-known-state and from alert-rule input; the raw row is queryable (FR-TEL-02 "never silently discard") |
| | `stuck` | Repeats the last value bit-identically for N samples | After N → `quality: suspect` on subsequent identical samples; cleared when the value moves. N **[DESIGN PARAMETER — proposed default 20 samples]**; `electronics-hardware-specialist` to confirm per sensor class |
| | `non-numeric` | Publishes `"value": "nan"` / null / string | Stored `implausible-rejected` with `reason: not_a_number`; ingest does not crash; no last-state update |
| | `unit-mismatch` | Publishes `unit: degF` | Stored `suspect` + event `unit_unexpected`; not converted (the backend never guesses units) |
| | `rate-of-change` | — | **R2.** Needs per-sensor response-time figures not present in the Phase D BOM; deferred rather than invented |

Every switch has a scenario in `sim/scenarios/` and a CI test; the 72 h soak scenario
chains `backend-only` (20 min), `edge-ungraceful` (30 min), `metric` stale (1 h), and a
`range` burst, and the pass condition is: zero rows in `telemetry_gaps` with status `open`
at the end, every `unfillable` row carrying a reason.

---

## 7. Edge vs cloud — what is non-negotiable even in a no-actuation release

Nothing in Phase 1 introduces software into a safety path. Recording the split so it is
inherited, not rediscovered:

| Lives at the edge (or in hardware) — non-negotiable | Lives in the backend — optional / advisory |
|---|---|
| De-energized relay state at boot/reset/flash on the ESP32 (§0.5) | Registry, lifecycle, commissioning audit |
| Offline detection of the room controller (broker LWT) and of rack nodes (bus timeout) | Mirror health badges with a coarser window; alerts |
| Command validation against the device's **own** advert; dedup ring; TTL expiry | Fast-path re-validation for UX (ADR-0001 rule 3) |
| Wall-clock and `seq` authority; outbound buffer | Gap detection and gap rows |
| `sys.identify` bounded by advertised `max_duration_s` at the device | Console rendering of "identifying…" |
| Leak / LSH-04 / CO2-safety state as **telemetry mirrors** of hard-wired interlocks | Alert rules on those mirrors, labelled *mirror, not safeguard* (classification §52-item 2) |

Anything in the right column can be absent and nothing physical changes — which in Phase
1 is trivially true, and the schema is shaped so it stays true when R2 adds actuators.

---

## 8. Flags for the HLD and downstream specialists

1. **Hardcode risks found** (must be discovered, not tabled):
   - "1 virtual rack (4 tiers)" → tier count from `profiles/*.json` (`tiers`), never a
     constant in `sim/` or `backend/`.
   - "Rack T/RH at tier 3" → it is the profile's `scope`, not a backend assumption.
   - Plausibility ranges → from the advert; backend class-fallback table is a fallback and
     records that it was used.
   - Staleness per metric → from `interval_s` in the advert with class floors; no
     "all racks report X" rule.
2. **`device-control-model.md` §4 amendments** to record in the HLD: explicit `<room>`
   segment; `<tier?>` moved to payload `scope`; `capability` retained leaf added; `event`
   type list. §6 example staleness values promoted to labelled design parameters.
3. **Outputs to name in the HLD:** schema v1 hardware review (§1.8); rack-link/0 as a
   named placeholder replaced by the R2 Modbus register map; relay-polarity bench check
   (§0.5) as a firmware bring-up acceptance item; `sim/` module with its own CI job.
4. **For `electronics-hardware-specialist`:** confirm T/RH sensor-class range for the
   advert; confirm relay module drive polarity; give per-sensor response-time figures if
   rate-of-change plausibility is wanted in R2; opinion on stuck-detection N per class.
5. **For `firmware-engineer`:** advert built from detected hardware; status-LED-only
   identify; JSON-lines rack-link/0 over UART; `link_seq` + `t_mono_ms` in every frame;
   no MQTT/TLS/NTP in the rack controller image.
6. **For `backend` (ingest):** `ts_ingest` on every row; staleness on `ts_ingest`;
   backfilled rows never touch live state or alert evaluation; gap table with status and
   reason; device_id vs topic cross-check.
7. **For `security-safety-reviewer`:** no fault-injection command exists in the device
   schema; unclaimed devices receive no commands at both layers; bench direct-MQTT (if
   used) has an isolated principal; `cmd`/`ack` QoS 1 minimum.
8. **FR-ID drift:** fault switches delivered in R1 under the release-plan lock while
   `requirements.md` files them under FR-SIM-02 (R2) — annotate, do not re-scope.

---

## 9. Design parameters — proposed defaults (none sourced; HLD to ratify)

| Parameter | Proposed default | Basis | Notes |
|---|---|---|---|
| Telemetry `interval_s` (T/RH, CO2, switches keepalive) | 30 s | none — volume/UX | Advertised per capability; sim and backend read it from the advert |
| Heartbeat interval | 30 s | none | In `health.heartbeat_s` so the backend can derive the window per device |
| `offline_after_s` | 90 s (3 missed heartbeats) | convention | Broker LWT and bus timeout are authoritative; this is the mirror window |
| `link_timeout_s` (virtual room controller → rack node) | 10 s | none | Real value belongs to the Modbus map (R2) |
| Staleness class floors | T/RH 120 s, CO2 300 s, switches 120 s | device-control-model §6 *example* | `max_age = max(3 × interval_s, floor)` |
| `ack_timeout_s` / `max_retries` | 5 s / 2 | none | Identify only; R2 classes may override |
| `ttl_s` (identify) | 60 s | none | Commands are never applied after expiry |
| `backend_wait_s` | 30 s | must exceed edge worst case (≈15 s) | Backend-side "unacknowledged" state, not an ack |
| Dedup ring per device | 32 cmd_ids | none | Memory-trivial on ESP32 and in sim |
| Outbound ring capacity (virtual room controller) | 6 h at nominal rate (~27 k msgs) | none | Reported in health; real controller is multi-day on disk (OQ-3 R11) |
| Broker `max_queued_messages` for the backend session | ≥ 2 × ring capacity | none | Infra task; undersizing is caught by gap detection, not hidden |
| Stuck-detection N | 20 identical samples | none | `suspect`, never rejected; per-class confirmation requested |
| `sys.identify.max_duration_s` | 60 s | none | Advertised; enforced at the device |

Nothing in this table is a physical or process safety threshold, and none of it may be
cited as one. The one threshold Phase 1 alert seeds may cite from `safety-rules.json` is
the CO2 5000 ppm alarm, and only as a **mirror** of the independent hardware monitor.
