# 04 — High-level design: phase-1-foundation

**Author:** hld-architect · **Date:** 2026-09-10 · **Pipeline:** `/ship` · **Branch:** `phase-1-foundation`
**Revision:** HLD-gate edits 2026-09-10 (brief §10.1 CONSISTENT; `05-security-review.md` PASS WITH FINDINGS — items F-1, F-5, F-7, F-4 applied here; F-2/F-3 carried to LLD in §10.3).
**Primary input:** `03-domain-brief.md` (status READY). Every **[HLD confirm]** item in it is
dispositioned in §10.1; none is overturned. **[proposed default, unsourced]** values are carried
with that label (§10.2); **[unknown]** items stay unknown (§10.3).
**Binding constraints read:** ADR-0001, 0002, 0003 (untouched), 0004, 0005, 0006, 0008, 0009;
`docs/architecture/system-architecture.md`; `docs/iot/device-control-model.md`;
`docs/security/security-architecture.md`; `docs/data/data-architecture.md`;
`docs/roadmap/release-plan.md` Phase 1 (LOCKED); `docs/requirements/requirements.md`;
`.github/agentic-rules/safety-rules.json` (2026-09-05); `docker-compose.prod.yml`;
`.github/workflows/build-test-deploy.yml`; trophic-contracts **pinned v0.2.0**
(`README`, `CHANGELOG`, `sensors/instrument-tags.md`, `capabilities/absent-by-design.md`,
`conventions/units.md`, `safety/interlocks.md`); OQ-3 decision brief (ADR-0007 reserved).
**ADRs opened by this HLD (status `proposed`):** ADR-0010, ADR-0011 — see §12.

**Scope guard.** Phase 1 actuates nothing. This design contains no desired-state store, no
schedule executor, no closed loop, no recipe, no OTA path, and no backend code path that can
publish a `cmd` for any class outside `sys.*`. Every safety-relevant reading in the brief
(§5.1 tiers, §5.3 bound rule, §5.4 relay GPIOs, §5.5 bands) is adopted as the reading that
takes no physical action and copies no invented number.

---

## 1. Design in one page

Phase 1 stands up the platform spine on a virtual facility and proves every seam later
releases reuse: identity, capability model, transport, storage, console, simulation, backup,
deploy. Four code subsystems plus two contracts of record:

```
                       docs/iot/schema/v1/           docs/api/openapi.v1.yaml
                       (JSON Schema + goldens +      (spec-first; both sides generate)
                        rack-link/0 framing)
                              │      │                          │
   firmware/rack-controller   │      │      sim/                │
   ┌──────────────────┐   rack-link/0  ┌────────────────────┐   │
   │ ESP32 skeleton   │──UART JSONL──▶│ virtual room ctl   │   │
   │ advert·telemetry │  (no MQTT,    │ (sole field MQTT   │   │
   │ sys.identify/ping│   no TLS,     │  client) + virtual │   │
   │ NO relay GPIO    │   no SNTP)    │  racks + I/O nodes │   │
   └──────────────────┘               │  + fault switches  │   │
                                      │  + simctl          │   │
                                      └────────┬───────────┘   │
                                     MQTT/TLS 8883, mTLS       │
                                               ▼               │
     ┌─────────── docker-compose.prod.yml (facility host) ──────┼──────────────┐
     │  mosquitto ──▶ backend/ (Go, one process)                │              │
     │   (per-CN      ├─ ingest   (schema·cross-check·dedup·gaps·plausibility·VPD)
     │    ACLs)       ├─ registry (facility tree, devices, lifecycle, capability index)
     │                ├─ alerts   (rule state machines, notify, SAFETY semantics)
     │                ├─ authz    (users, roles, TOTP, sessions, audit)
     │                ├─ backup   (scheduled + export/import, encrypted bundles)
     │                └─ api      (OpenAPI read/write surface + serves console bundle)
     │                         │                                                │
     │  timescaledb ◀──────────┘         frontend/ (Vue 3, static bundle) ◀─────┘
     └────────────────────────────────────────────────────────────────────────────┘
```

The room controller (virtual in Phase 1) is the **only** MQTT client on the field side
(ADR-0004 point 1). Rack controllers speak `rack-link/0` to it and hold no broker
credential. The backend is never something the room depends on (ADR-0001): stopping it
stops nothing in `sim/` and would stop nothing in a real room.

---

## 2. Subsystem map and boundaries (ADR-0008)

`sim/` joins `backend/`, `frontend/`, `firmware/` as a **fourth subsystem** bound by
ADR-0008 rules 1–4. This is an application of ADR-0008 (nothing is split), not a
supersession; `docs-writer` adds `sim/` to the CLAUDE.md boundary list at ship time (§13).

| Subsystem | Language / manifest | Contents (Phase 1) | Couples to others only via |
|---|---|---|---|
| `backend/` | Go, `backend/go.mod` (ADR-0006) | One binary, modules `authz`, `registry`, `ingest`, `alerts`, `backup`, `api`; subcommands `serve`, `migrate`, `export`, `restore`, `seed` | MQTT schema v1 (broker side); OpenAPI v1 (console side); Postgres |
| `frontend/` | TypeScript, Vue 3 + Vite, `frontend/package.json` (ADR-0009) | Console v0: Login, Home, Grow, Devices (+ commissioning wizard), Alerts, Settings (Users, Backup & export, System, Notifications section, Safety envelope read-only, Facility metadata) | OpenAPI v1 generated client **only**; never MQTT |
| `firmware/rack-controller/` | C, ESP-IDF, `firmware/rack-controller/CMakeLists.txt` | ESP32 skeleton v0: capability advert built from detected hardware, register-image sample path, `rack-link/0` JSON-lines over UART, `sys.identify` (status LED only) and `sys.ping`; **no MQTT/TLS/SNTP; relay GPIOs compiled out** | `rack-link/0` framing (`docs/iot/schema/v1/rack-link-0.md`) |
| `firmware/room-controller/` | — | `README.md` only: "empty until OQ-3 closes as ADR-0007; the virtual room controller in `sim/` is a simulator, not this" | — |
| `sim/` | Go, `sim/go.mod` (separate module; **not** a backend package) | `cmd/virtual-room` (virtual room controller: MQTT client, bridge, outbound ring, health-on-behalf, cmd router), `cmd/simctl`, `internal/vdevice`, `internal/racklink` (in-process + serial adapters), `internal/faults`, `profiles/*.json` (data), `scenarios/*.yaml` | MQTT schema v1; `rack-link/0`; `simctl` local control surface (never the device `cmd` schema) |
| `docs/iot/schema/v1/` | JSON Schema + golden messages + `rack-link-0.md` | **Contract of record firmware ↔ sim ↔ backend** | — |
| `docs/api/openapi.v1.yaml` | OpenAPI 3.1 | **Contract of record backend ↔ frontend** | — |

### 2.1 How each boundary is enforced (build time, not review time)

| Rule (ADR-0008) | Enforcement |
|---|---|
| 1. OpenAPI is the contract of record | **Spec-first.** `docs/api/openapi.v1.yaml` is hand-authored in the LLD. `backend/` validates every route against it in tests (request/response schema validation middleware in test mode; a test fails if a served route is absent from the spec or vice versa). `frontend/` commits its generated client (`openapi-typescript` + `openapi-fetch`, ADR-0009) and CI fails if regeneration produces a diff. No TypeScript type is written by hand for an API payload. |
| 2. Firmware ↔ backend couple only via the versioned MQTT schema | Firmware speaks no MQTT in Phase 1 at all; it emits v1 payload *objects* over `rack-link/0`. `firmware/`, `sim/`, and `backend/` each run a **conformance test** that parses and re-emits the golden messages under `docs/iot/schema/v1/` (loaded from the docs path in CI; if a subsystem is ever extracted it pins a published schema version instead). No Go type for a payload is shared by path: `sim/` and `backend/` each own their struct definitions and both are checked against the same schema. |
| 3. No cross-directory imports | A `boundary-check` CI job: (a) `backend/go.mod` and `sim/go.mod` may not `require` or `replace` each other; (b) no Go import path under `backend/` references `.../sim/` or vice versa; (c) `frontend/` ESLint `no-restricted-imports` for `../backend`, `../sim`, `../firmware`; (d) `firmware/` CMake component search paths are confined to `firmware/rack-controller/`. The job greps for any relative path crossing a subsystem root in source files and fails on a hit. |
| 4. Own manifest, independently runnable CI job | `detect` job probes `backend/go.mod`, `frontend/package.json`, `firmware/rack-controller/CMakeLists.txt`, `sim/go.mod`; each subsystem job runs alone with `working-directory` set. Console bundle reaches the backend image by **multi-stage Docker build with repo-root context** (stage 1 builds `frontend/`, stage 2 builds `backend/`, final image places `dist/` at `/srv/console` which the backend serves from a configured directory). That is artefact composition at image build time; the backend never imports or embeds frontend source. |

**The `sim/profiles/esp32-skeleton-v0.json` file is a captured advert**, not shared source:
it is regenerated by pointing `virtual-room`'s serial adapter at a real board and recording
what arrives, and CI diffs it against the firmware conformance test's emitted advert.

---

## 3. Contracts of record

### 3.1 MQTT topic grammar and payload schema v1 (`docs/iot/schema/v1/`)

Adopts IoT §1 as reconciled by brief §2. This **amends `device-control-model.md` §4**
(`docs-writer` at ship time) and changes nothing in ADR-0004: the `trophic/<org>/<site>`
root, per-device leaves, retained state, `cmd`/`ack` never QoS 0, and the room controller as
the sole field client are untouched. ADR-0004 point 3's inline grammar is illustrative; §4
as amended is normative.

```
trophic/<org>/<site>/<room>/<zone>/<device>/<leaf>
  <zone>   rack-NN | room          (tier is payload `scope`, not a topic segment)
  <device> registry device_id      rc-01 … rc-11, rmc-01, io-terrace, io-skid, io-room
  <leaf>   capability      retained QoS1     profile advert (+ profile_sha256)
           state/health    retained QoS1     heartbeat; LWT on the room controller's own topic
           state/reported  retained QoS1     defined, unused in Phase 1
           state/desired   retained QoS1     defined, unused in Phase 1
           telemetry/<metric>      QoS1      one sample per message
           cmd / ack               QoS1      never QoS 0
           event                   QoS1      boot | claim | link_gap | buffer_overflow | schema_unsupported | clock_anomaly
```

Envelope (every message): `v`, `msg`, `org`, `site`, `room`, `zone`, `device_id`, `boot_id`,
`seq`, `ts_source`, `t_mono_ms?`, `link_seq?`, `clock ∈ {ntp, rtc, unsynced}`, `sim`,
`backfill`. Version lives in `v`, never the topic root. Schema files: one JSON Schema per
`msg` type, one for the capability profile (ADR-0002 descriptor, `schema_version: "1.0"`),
one enumerating `reason_code` v1 (IoT §3.4, closed set incl. reserved R2 codes), one
enumerating the closed `sys.*` action set (ADR-0010 point 4), one enumerating allowed
`unit` strings (`degC`, `pct`, `ppm`, `umol_mol`, `kPa`, `mS_cm`, `L_min`, `m3_h`, `m_s`,
`Hz`, `bool` — the four not in contracts `units.md` are flagged for the hardware review,
§13). Golden example messages for every type and every profile.

Two rules carried from the brief that the schema README states in bold:

- **ACL wording (AC-ADR-0005a restated):** a principal cannot publish outside its ACL
  prefix; the room-controller principal's prefix is its room (`…/<room>/#`); a bench/direct
  principal's prefix is its own device (`…/<zone>/<device>/#`); rack nodes hold no credential.
- **Ingest cross-check:** payload `device_id` vs topic prefix mismatch → stored `suspect` +
  security event; never dropped, never trusted.

### 3.2 `rack-link/0` (ESP32 skeleton ↔ room controller) — explicit placeholder with a sunset

Adopted per brief §1 **[HLD confirm → adopted]**:

1. Newline-delimited JSON frames over UART (USB-serial on the bench; RS485 transceiver when
   fitted) carrying the v1 payload objects **minus** `org/site/room/ts_source/clock`, **plus**
   `link_seq` and `t_mono_ms`. The room controller stamps `ts_source`, `clock`, `seq`,
   `boot_id` and publishes.
2. The skeleton's sample path is **register-image shaped**: latest value per channel + `seq`
   + change flags, produced on a timer, serialised to frames. R2's swap to the Modbus
   register map is a framing change, not a redesign.
3. The image contains **no MQTT, TLS, or SNTP**; no rack-controller class device holds a
   broker credential.
4. Direct-MQTT-from-ESP32 is **not built**. If bring-up needs it: compile-time bench flag,
   distinct broker principal confined to `…/<zone>/<device>/#`, ECC P-256 mTLS per
   ADR-0011, deleted before R2.
5. **Sunset:** replaced by the Modbus register map (contracts "not published"; H10) in R2 —
   named as a **trophic-contracts v0.3.0 dependency for R2**. ADR-0007 (OQ-3), when drafted,
   must reference `rack-link/0` so this placeholder cannot silently become the production
   link. No new ADR: this applies ADR-0004 point 1.

The `RackLink` interface in `sim/internal/racklink` has two implementations — in-process
channel (virtual racks) and serial (real skeleton) — so "≥1 real rack controller if hardware
ready" is the same acceptance test with one adapter swapped. **Fingerprint caveat (brief
§8.1):** a real board's fingerprint (e.g. eFuse MAC) travels in the rack-link advert and is
forwarded unaltered; the room controller cannot attest it. Fine for Phase 1 (virtual); an
open security item for the real bus in R2, flagged to `security-safety-reviewer` (§14).

### 3.3 Capability profile semantics (ADR-0010, as amended at the HLD gate)

Profile = `capabilities[]` (**hardware inventory**, each with `hardware: present|optional`)
+ `commands[]` (**classes/actions this firmware version accepts**, `{class, actions[]}`),
with `commands ⊆ capabilities` a validation error **without exemption**. The `sys.*`
namespace is a **closed set of actions of the `control.compute` capability** — not a
capability class of its own — so every controller-class device, which already advertises
`control.compute {role}`, satisfies the invariant when it lists
`commands = [{class: control.compute, actions: [sys.identify, sys.ping]}]`. A command
message carries `capability: control.compute, action: sys.identify`. The set is enumerated
in `docs/iot/schema/v1/`; admission requires that the action drive no output (relay, PWM,
0–10 V, LED, pump, valve, doser, HVAC) and write no stored schedule or setpoint — the
property that justifies its exemption from interlock/envelope checks — and adding a member
requires amending ADR-0010. A class advertised but not in `commands` (or a `sys.*` action
outside the closed set) acks `REJECTED{action_not_supported}` at the edge; a class not
advertised acks `REJECTED{capability_not_advertised}`. Full statement and alternatives in
ADR-0010.

**Phase 1 profile set** (all `sim: true` except the captured skeleton advert):

| Profile (`sim/profiles/`) | `capabilities[]` | `commands[]` |
|---|---|---|
| `rack-controller-std-v0.json` (×11 soak; ×1 dev; parametrised to ≥13 for the ≥50-zone load run) | `control.compute{role: rack}`, `sense.temp_rh @ rack-NN/tier-3` (T ±0.3 °C, RH ±2 % class accuracy), `sense.leak{LD-NN}`, `sense.water_level{LSH-04, kind: switch}`, `airflow.tacho ×4` (metric `fan_tacho_hz`, unit `Hz`), `light.state_report ×4` (sim-declared), `valve.control{failsafe_state: "CLOSED"} ×4` (fill), `valve.control{failsafe_state: "OPEN"} ×4` (drain), `irrigation.flood ×4`, `airflow.speed ×4`, `light.dim ×4` — `tiers: 4` from the profile, never a constant | `[{class: control.compute, actions: [sys.identify{max_duration_s: 60}, sys.ping]}]` |
| `room-controller-v0.json` (`rmc-01`) | `control.compute{role: room}`, `sense.temp_rh @ room` (supply + return), `sense.co2 @ room` (enrichment NDIR, range 400–5000 ppm, Phase D SCD41 class), `sense.co2 @ room {role: safety-mirror}` (**range omitted** → backend class fallback, recorded as fallback, §4.3) | `[{class: control.compute, actions: [sys.identify, sys.ping]}]` |
| `io-terrace-v0.json`, `io-skid-v0.json`, `io-room-v0.json` (D-10) | `control.compute{role: io-node}`; terrace: `sense.water_temp{TE-01}`, `sense.water_level{LT-02}`, `sense.ph`, `sense.ec`; skid: `sense.water_temp{TE-02}`, `sense.ec{AT-03}` (mS/cm), `sense.ph{AT-04}`; room: floor-flood/E-stop **only if** the hardware program confirms they are read by the room I/O node — otherwise omitted (**[unknown]**, not invented) | `[{class: control.compute, actions: [sys.identify, sys.ping]}]` |
| `esp32-skeleton-v0.json` (captured) | Whatever the board detects: `control.compute{role: rack}`, `sense.temp_rh` iff SHT-class found on I²C, `sense.leak`/`sense.water_level` iff dry contacts configured, actuator *counts* from an empty-by-default board descriptor (no pin mapping), optional `sys.heap_free` diagnostic metric on `control.compute` (D-16) | `[{class: control.compute, actions: [sys.identify, sys.ping]}]` |

`failsafe_state` strings are copied **verbatim** from `safety-rules.json
valve_and_actuator_fail_safe_states` (`rack_fill_solenoids: "CLOSED …"`,
`rack_drain_solenoids: "OPEN …"`; the profile carries the leading token, the registry keeps
the key path). Nothing from `absent-by-design.md` is advertised or rendered. **Standard
profile only**; no Pro profile is invented (brief §3).

### 3.4 OpenAPI v1 (`docs/api/openapi.v1.yaml`)

Surface per UX appendix, org-resolved from session (no client-supplied org id, ADR-0005).
Two cross-cutting types the LLD must define once: the **value envelope** (`value, unit,
provenance ∈ {measured, calculated, estimated, operator_entered}, ts_source, time_basis,
age_s, stale, staleness_max_age_s, quality, formula_version?, inputs?, calibration_id?, sim,
instrument_tag?`) consumed by one Vue component so no screen renders a bare number; and the
**as_of** response header/field every read carries. Writes in Phase 1 are all non-actuating:
auth, claim/verify/assign/activate, `POST /devices/{id}/commands` (**`sys.identify` only**;
the server rejects any other class before touching the broker), commissioning confirm,
alert ack/resolve, `PATCH /alert-rules/{id}` (non-SAFETY zone overrides; console read-only in
v0, D-7), users/TOTP, zone active/idle with reason, facility metadata (FR-FAC-05 +
`commissioning_ppfd`), backups export/import-validate, notification channels.

---

## 4. Data and control flow

### 4.1 Telemetry path

```
ESP32 skeleton ──rack-link/0──┐
virtual racks  ──in-process───┼─▶ virtual room controller ──MQTT/TLS 8883 (mTLS, ACL room prefix)──▶ Mosquitto
virtual I/O nodes ─in-process─┘      · stamps ts_source/clock/seq/boot_id                                │ persistent session,
                                     · health on behalf of nodes (bus_timeout after link_timeout_s)      │ max_queued_messages
                                     · outbound ring (file-backed, 6 h) → replay backfill:true            ▼
                                                                                          backend/ingest (one MQTT client, service cert)
                                                                                                          │
   ①TLS+ACL ②schema(v,msg) ③device_id↔topic cross-check ④registry resolve ⑤dedup(device_id,boot_id,seq) ⑥gap detect
   ⑦plausibility gates ⑧write raw row (+ts_ingest, quality, backfill) ⑨live path: last-known-state → derived (VPD) → alert eval
   ⑩health machine update
                                                                                                          ▼
                                                                                             TimescaleDB ──▶ api ──▶ console
```

Ingest step semantics (all **flag, never drop**):

| Step | Rule |
|---|---|
| ② | Unknown `v`/`msg`/profile `schema_version` newer than known → stored raw, `event{schema_unsupported}`, device visible with badge; never a crash (ADR-0002 rule 3). |
| ③ | Payload `device_id` ≠ topic `<device>` → row `quality: suspect`, `quality_reason: topic_mismatch`, security audit row. |
| ④ | First `capability` (or `health`/`event`) from an unknown `device_id` creates an **UNCLAIMED** registry row. Telemetry from an unclaimed device is stored marked `unclaimed`, joined to no zone. A `sim` flag differing from the claimed value → profile diff → CLAIMED + security event. |
| ⑤ | `(org, device_id, boot_id, seq)` already stored → **idempotently ignored** (D-11). Re-delivered QoS1 messages and clock-frozen repeats therefore never double-store. |
| ⑥ | Per `(device_id, boot_id)`: `seq` jump > 1 → `telemetry_gaps` row `{seq_from, seq_to, detected_at, status: open, cause}`; cause by correlation (§4.5). |
| ⑦ | In order: unit ∈ allowed list (else `suspect`, `event{unit_unexpected}`, **never converted**); numeric (else `implausible-rejected{not_a_number}`); range from the **advert** first, backend **class fallback** second (fallback use recorded on the row); stuck-value N identical → `suspect{stuck}`; `ts_source` non-advancing vs advancing `ts_ingest` → `suspect{clock_anomaly}` + `time_basis: ingest`. Rate-of-change: **R2** (no per-sensor figures). Backend may only **downgrade** the device's verdict. |
| ⑧ | Every row carries `ts_source`, `ts_ingest`, `time_basis ∈ {source, ingest}` (`ingest` when `clock: unsynced` or clock anomaly), `seq`, `boot_id`, `quality`, `quality_reason`, `backfill`, `calibration_id` (nullable now). |
| ⑨ | **Only** rows with `backfill: false` and `quality ≠ implausible-rejected` touch last-known-state, derived series, and alert evaluation. Backfilled rows are history: they fill gap rows and rollups, nothing else. |
| ⑩ | See §4.2. |

**Staleness runs on `ts_ingest`** (D-12): `max_age(metric) = max(3 × interval_s from the
advert, class floor)`; floors T/RH 120 s, CO2 300 s, switches/state reports 120 s,
water-side classes 120 s (this HLD's floor, §10.2). Display age (`age_s` in the value
envelope) is server `as_of − ts_effective` where `ts_effective = ts_source` if `time_basis:
source` else `ts_ingest`; the `stale` flag is always the `ts_ingest` verdict. A metric the
profile does not advertise cannot be stale.

**VPD (FR-TEL-03).** Computed by the backend from the same T/RH sample pair — pairing key
`(device_id, scope, ts_source)`; no pair → no VPD row (never zero, never cross-node).
Formula `es(T) = 0.6108·exp(17.27·T/(T+237.3))` kPa, `VPD = es(T)·(1 − RH/100)`; cited as
**external standard — FAO Irrigation & Drainage Paper 56, Eq. 11 (Tetens form)**, not a
project-sourced value. Version `vpd_air.tetens_fao56.v1`, `leaf_temp_assumption:
equal_to_air`; output quality = worst input; RH > 100 % → negative VPD stored
`implausible-rejected`, never clamped; sensor accuracy class stored alongside so the
≈ ±0.06–0.07 kPa propagated uncertainty is visible (electronics Q2). Unit-test vectors:
botany §2.3.

**DLI (D-4, ship minimal).** Tier attribute `commissioning_ppfd` (nullable,
`operator_entered`, dated; editable with FR-FAC-05 metadata). `DLI = PPFD × lights-on hours
from light.state_report × 3600 / 1e6`, computed per facility-local day, `formula_version`
`dli_est.state_x_ppfd.v1`, provenance **`estimated`**, **null when no PPFD, never zero**,
input chain on tap, no DLI rule or badge (`dli_mol_m2_day` is TBD). Virtual racks only.

### 4.2 Device state machines (IoT §2, brief D-9/D-12)

**Lifecycle (registry-owned, audited per transition with actor):**

```
(none) ─first advert─▶ UNCLAIMED ─claim─▶ CLAIMED ─assign─▶ COMMISSIONED ─activate─▶ ACTIVE
                          ▲                 ▲  (verify + identify SUCCESS + on-site confirm recorded)   │
   fingerprint mismatch ──┘ QUARANTINED     └──── profile diff after activation (keeps telemetry) ──────┘
                                                                            operator retire ──▶ RETIRED
```

- UNCLAIMED is untrusted: no command is ever published to it (backend rule) and the edge
  rejects with `device_unclaimed` in case the backend rule is bypassed.
- **Trust chain:** a bus node bridged by the room controller cannot be claimed until its
  parent (`rmc-01`) is CLAIMED or later; its adverts are stored meanwhile.
- `sim` is fixed at claim; a later flip is a profile diff + security event.
- QUARANTINED is shown as a claim state with reason; revoke waits for R2 (D-15/F-8).

**Health (derived, two axes, one badge; precedence OFFLINE > FAULT > DEGRADED >
STALE-TELEMETRY > ONLINE):**

- Connectivity: `UNKNOWN → ONLINE` on first message; `→ OFFLINE` on no heartbeat **and** no
  telemetry for `offline_after_s`, or LWT, or `health{offline}`, or parent OFFLINE
  (`offline, inherited`); back to ONLINE when a heartbeat resumes.
- Telemetry axis per advertised metric: `FRESH → STALE` at `max_age` on `ts_ingest`.
  **STALE requires ONLINE** — an OFFLINE device's metrics are not STALE, which is what makes
  the root-cause collapse (one alert row per offline rack, not four stale rows) fall out of
  the machine rather than out of alert logic.
- `DEGRADED` self-reported (`clock_unsynced`, `buffer_dropped`, `sensor_init_failed`,
  `command_timeout`); `FAULT` reserved for `event{type: fault}` (R2). Both verified by unit
  tests on the machine (AC-DEV-04b).
- Alerts: STALE-TELEMETRY → one OPERATIONAL per device (FR-TEL-04); OFFLINE → **one
  OPERATIONAL "device offline" per device** (D-9 **[HLD confirm → adopted]**), affected
  zones listed; a node whose OFFLINE is inherited raises no row of its own — it appears in
  the parent's `affected` list (`suppressed_under`). Citations: "design parameter, HLD
  §10.2", never `safety-rules.json`.
- Production note carried forward: on the real bus the room controller's Modbus timeout is
  authoritative; the backend window is a mirror. Nothing in the room waits on the backend.

**Zone status (FR-FAC-03):** `active`/`idle` operator-set with reason (audited); `offline`
derived when every device serving the zone is OFFLINE or STALE-TELEMETRY, reason naming the
device(s); clearing restores the prior operator-set state.

### 4.3 Plausibility bounds and the bound-vs-threshold invariant

Ranges come from the advert; the backend class-fallback table is used only when a profile
omits `range`, and the row records that it fell back. **Seeding invariant (brief §5.3,
[HLD confirm → adopted]):** *for any channel carrying an interlock-mirror rule, the effective
plausibility upper bound must exceed the mirror threshold; rule seeding fails otherwise.*
This is why the safety-NDIR profile omits `range` (its real part is **[unknown]**) and
falls back to the 0–10 000 ppm class value, and why the water-temperature class fallback
(§10.2) is set above 28 °C. Seeding runs at migration/seed time and at every profile change
that alters a range; a violation is a startup/seed error, never a silently disabled rule.

### 4.4 Command path — `sys.identify` and the REJECT paths

```
console ─▶ backend api: authz (technician|admin); device ∈ {COMMISSIONED (guided test), ACTIVE};
           class ∈ allow-list {sys.identify, sys.ping}; class ∈ registry profile.commands;
           params ≤ advertised max_duration_s
        ─▶ ingest module's single command publisher: cmd{cmd_id (ULID), capability, action, params,
           issued_at, expires_at = +ttl_s, actor, cmd_seq} on …/<zone>/<device>/cmd  (QoS1)
virtual room controller: re-validates against the device's OWN advert; claimed?; not expired?;
           cmd_id in dedup ring (32) → replay stored ack, duplicate:true
   ├─ REJECT → ack{REJECTED, reason_code}
   └─ ACCEPT → forward (rack-link / in-process); wait ack_timeout_s; retry same cmd_id ≤ max_retries;
              then ack{TIMEOUT} + health DEGRADED{command_timeout}; OFFLINE target → park until expires_at → ack{EXPIRED} + event
backend: command record {issued, acked, result, latency}; no ack by backend_wait_s → state "unacknowledged"
         (the backend never synthesises an ack on the ack topic)
```

`sys.identify` is a **no-op with an observable, side-effect-free indication** — status LED
on the controller board / visible `identify` flag on a virtual device — bounded by the
advertised `max_duration_s` at the device. It never touches a relay channel, the 0–10 V
pair, fan PWM, or a grow LED. The console shows the reported `identify` state so the
round-trip is verifiable even if the LED cannot be seen through the enclosure (electronics
Q4). `sys.*` commands are not subject to interlock/envelope checks (they actuate nothing)
but are subject to `device_unclaimed`, `expired`, `stale_sequence`, `busy`,
`params_out_of_range`.

**REJECT paths exercised in CI (the ADR-0002 "capability absent is first-class" test):**

| Case | Origin | Expected |
|---|---|---|
| Guided test against an UNCLAIMED device | console → backend | 409 server-side; nothing published |
| Any class ∉ allow-list from the console | console → backend | 400; nothing published (AC-DEV-03e static/contract check also asserts the publisher's allow-list) |
| `co2.enrich` / `dosing.fertigation` to a virtual rack | harness test client (CI-only principal) → broker → edge | `ack{REJECTED, capability_not_advertised}` |
| `light.dim set 70` to a virtual rack (advertised, not in `commands`) | harness test client | `ack{REJECTED, action_not_supported}` |
| `sys.identify duration_s: 600` | console | edge `ack{REJECTED, params_out_of_range}` (backend fast-path also refuses) |
| Duplicate `cmd_id` | harness | stored ack replayed, `duplicate: true` |

**Commissioning round-trip (AC-DEV-03a/UX §5, two-party):**

| Step | Transition / record | Backend rule |
|---|---|---|
| 0 Prepare | device appears UNCLAIMED (poll) | first advert creates the row |
| 1 Claim | UNCLAIMED → CLAIMED | technician/admin only; last-4 fingerprint typed must match; mismatch with a known identity → QUARANTINED, wizard stops |
| 2 Verify | verification record | parsed profile vs expected counts from commissioning guidance (4 fill NC, 4 drain NO, 4 fans, 4 LED pairs, 1 T/RH — instrument-tags); each `valve.control.failsafe_state` cross-checked **verbatim** against `safety-rules.json valve_and_actuator_fail_safe_states`; mismatch = blocking fault (cannot activate); unknown classes shown "recorded, not rendered" |
| 3 Assign | CLAIMED → COMMISSIONED | zone rows linked (1 tier = 1 zone default); instance remap allowed; existing controller on the rack = conflict requiring explicit retire |
| 4 Identify | command record + on-site confirmation record (`confirmed_by`) | `sys.identify`; REJECTED/TIMEOUT/"No" keeps the wizard on step 4; retry = new `cmd_id` |
| 5 Activate | COMMISSIONED → ACTIVE; zone `active` or `idle{no schedule}` | requires a SUCCESS ack + confirmation on the current assignment |

Six audit rows minimum, actor per row; `operator`/`viewer` refused at claim (403).

### 4.5 Sequence, gaps, backfill (NFR-REL-02 "architecture shape", D-12/D-13)

- `seq`/`boot_id` are the **publisher's** (room controller) per device stream, restarting at 1
  under a new `boot_id`, persisted with the outbound ring so a publisher restart never
  silently reuses numbers. Rack-link nodes add `link_seq` + `t_mono_ms`; a bus-level loss is
  `event{link_gap}` and is **unfillable by construction** in Phase 1 (no buffer on the
  ESP32).
- Gap record = `(org, device_id, boot_id, seq_from, seq_to | ts window, cause, detected_at,
  status ∈ {open, filled, unfillable}, reason)`. Cause by correlation: inside a recorded
  ingest outage window → `backend-down`; broker LWT / health offline → `device-offline`;
  broker unavailable → `broker-down`; `buffer_overflow` event covering the range →
  `unfillable{buffer_overflow}`; `link_gap` → `unfillable{link_gap}`; else `unknown`
  (permitted; explained in the soak report).
- The backend records its own **ingest outage windows** (start on connect after an absence
  or on start-up, end at reconnect) — these are the "gap records with cause backend-down"
  of product §6.1 even when the broker's persistent session held every message and no `seq`
  gap exists.
- **In Phase 1:** gap detection + recording; push-on-reconnect replay from the virtual room
  controller's file-backed ring (`backfill: true`, `seq` order; `filled` when covered);
  broker persistent session for the backend principal sized ≥ 2 × ring capacity so a
  backend-only outage needs no device replay. **R2:** pull backfill (`sys.backfill`) from
  the real room controller's local store (OQ-3 / ADR-0007).
- One OPERATIONAL "telemetry gap" alert per device while any `open` gap row exists;
  auto-resolves when the row becomes `filled` or `unfillable{reason}` (the gap is then
  accounted).

### 4.6 The 72 h soak scenario shape (D-5, IoT §6)

`sim/scenarios/72h-soak.yaml`: 11 racks × 4 tiers + `rmc-01` + three I/O nodes at 30 s;
steady state 22 °C / 65 % RH (inside all three climate bands, never a rectangle corner);
diurnal shape per botany §4.2 (18 h on / 6 h off staggered; RH may drift to 71–73 % for tens
of minutes at night so the RH ADVISORY fires and clears; tachos constant day and night; EC
constant-plus-noise labelled `unsourced-sim-placeholder`). Scripted: `backend-only` outage
20 min, `edge-ungraceful` 30 min, `metric` stale 1 h, one `range` burst, ≥1 `offline` rack.
Pass: zero `open` gap rows at the end, every `unfillable` row with a reason, 1 m/1 h
aggregates complete, ingest lag non-monotonic, **no SAFETY alert raised by the soak
fixture**. The test-only `override` switch (§4.7) runs in its own scenario, never in the
soak.

### 4.7 Fault switches (`simctl` / scenario only — never the device `cmd` schema)

Taxonomy per IoT §6 (D-11): `offline{device | edge-ungraceful | edge-graceful}` (+ the
infra switch `backend-only`), `stale{metric | all | clock-frozen}`,
`implausible{range | stuck | non-numeric | unit-mismatch}` (rate-of-change: R2), and the
fourth, **test-only `override`** (D-5): sets a metric's value (e.g. safety-NDIR → 5 200 ppm
for a few minutes) so FR-ALR-03 / AC-ALR-03a can be exercised end-to-end; injected values
are labelled `is_test_injection` in alert history; own CI scenario and e2e, not the soak.
There is no fault-injection command in schema v1; a real device cannot receive one.

---

## 5. Storage model (HLD level; DDL is LLD)

Single PostgreSQL 16 + TimescaleDB instance (ADR-0004). **Every table carries `org_id`**;
the data-access layer takes org scope from the request context and no method exists without
it (ADR-0005; coverage gate §9). Forward-only migrations, versioned; bundle carries the
schema version.

| Area | Entities (key content) | Kind |
|---|---|---|
| Tenancy / facility | `orgs`, `sites`, `rooms`, `racks`, `tiers`, `zones`, `zone_tiers`; lifecycle `installed\|active\|maintenance\|retired`; FR-FAC-05 metadata (`position_ref`, `tier_height_mm`, `notes`, `doc_links`, `commissioning_ppfd{value, entered_by, entered_at}`); zone `status{active\|idle\|offline}`, `status_reason`. Structure created by `backend seed` from a **versioned seed file** applied at deploy or via API (D-3); admins edit metadata only. **No X/Y rack coordinates seeded from the OCR'd floor plan** (OQ-4). | relational |
| Identity | `users` (email, argon2id hash, role, disabled), `totp_secrets` (encrypted at rest with the session/secret key), `sessions` (opaque id, idle + absolute expiry) | relational |
| Registry | `devices` (device_id, parent_device_id, descriptive type, hw_rev, fw_version, lifecycle, claim fields, `hw_fingerprint`, `sim`, `credential_type`, `credential_fingerprint`, `acl_prefix`, `profile_sha256`); `device_profiles` (JSONB descriptor + `schema_version`, append-only per advert change); `capability_index` (device, class, scope, instance, tag, params, `hardware`, `in_commands`) — makes "which devices advertise `sense.co2`" plain SQL; `device_assignments` (instance → zone); `commissioning_steps` (append-only); `device_health_current` | relational |
| Telemetry | `telemetry_raw` hypertable: (org, device_id, metric, scope, tag, `ts_effective`, `ts_source`, `ts_ingest`, `time_basis`, seq, boot_id, value / value_bool, unit, quality, quality_reason, range_source ∈ {advert, class_fallback}, calibration_id, backfill, unclaimed) — uniqueness on (org, device_id, boot_id, seq); `telemetry_derived` hypertable (VPD, DLI: value, formula_version, inputs ref, provenance, quality); `device_health_events` hypertable; `device_events` hypertable (boot, claim, link_gap, buffer_overflow, schema_unsupported, clock_anomaly, quality transitions); `last_known_state` (zone/device × metric → value envelope); `telemetry_gaps`; `ingest_outages` | hypertables + hot tables |
| Rollups | Continuous aggregates 1 m, 1 h, 1 d per (zone/device, metric): min/max/avg/count, `n_suspect`, `n_rejected`; **exclude `implausible-rejected`, include `suspect` counted**. Raw retention policy ops-configurable (NFR-DAT-03 target ≥ 24 months), export-before-rollup possible; chart cut-over raw ↔ 1 m ↔ 1 h documented in the LLD. | continuous aggregates |
| Alerts | `alert_rules` (metric/class, comparator, threshold + unit, `duration_s`, hysteresis, scope kind, severity, enabled, `is_interlock_mirror`, `source_citation{key_path, source_string}` or `"design parameter, HLD §10.2"` or `"operator-configured, unsourced"`, `protects_text`, `action_codes[]`, `editable`); `alert_rule_overrides` (per zone, labelled operator-configured); `alert_eval_state` (rule × scope: `idle\|breaching{since}\|raised`, last value — **persisted**, so a restart mid-condition does not re-raise); `alerts` (raised/notified/ack{by, at, action_code, note}/resolved{by\|auto, at, reason}, current value snapshot, `affected_zones[]`, `suppressed_under`, `is_test_injection`); `alert_notifications`; `notification_channels` (per severity; SAFETY additive only) | relational |
| Commands | `commands` (cmd_id, device, capability, action, params, actor, issued_at, expires_at, cmd_seq, backend_state ∈ {issued, acked, unacknowledged}, ack{result, reason_code, latency_ms, state_after, received_at}); `identify_confirmations` | relational |
| Audit | `audit_log`: **append-only** — the application DB role has INSERT/SELECT only, and a `BEFORE UPDATE OR DELETE` trigger raises unconditionally (AC-USR-02a tests both). Rows: actor{kind ∈ user\|system\|device, id}, action, target, before/after JSONB, request id. Security events (topic mismatch, sim flip, fingerprint mismatch, ACL refusals seen) are audit rows with `actor.kind = system`, category `security`. | relational, append-only |
| Backup | `backup_runs` (scheduled\|on-demand, started/finished, status, bundle path, size, sha256, manifest JSONB), `import_runs` (validation report, policy, result) | relational |
| Platform health | `platform_health` small hypertable (ingest lag, broker connected, storage bytes, restart markers) feeding Settings → System and Home's Platform tile | hypertable |

### 5.1 Backup bundle shape and the clean-stack restore drill (D-6)

Bundle = one archive, **encrypted at rest** with authenticated symmetric encryption under
the `backup_key` compose secret (key never inside the bundle; security §1/§7):

```
trophic-export-<org>-<UTC ts>.tar.age   (encryption tooling is LLD; the property is binding)
  manifest.json      bundle_schema_version, app_version, db_schema_version, exported_at, org,
                     content types with COUNTS, files[] with sha256, raw_window_days,
                     absent_entity_types: [recipes, batches, inventory, orders]  (declared, not omitted)
  manifest.sha256
  entities/<table>.jsonl        every relational table above, org-scoped
  audit/audit_log.jsonl
  telemetry/rollups_{1m,1h,1d}.jsonl
  telemetry/raw_<from>_<to>.jsonl   window per manifest (default §10.2)
```

**Restore/import is an operator CLI in v0** (`backend restore --validate | --policy
import-as-new|replace`), because the locked drill restores into a **clean stack with fresh
volumes** where no user account exists to log into a console until the bundle restores
them. The console shows export-now (J7), bundle list, validation report, drill status.
Drill (documented in `docs/ops/restore-drill.md`, the runbook the e2e follows): fresh
volumes → `backend migrate` → `restore --validate` (schema version + every checksum; a
tampered byte refuses naming the file; incompatible schema refuses with reason) → `restore
--policy import-as-new` → counts equal manifest; audit intact and still append-only;
rollups restored → backend **re-materialises broker ACLs from the restored registry** (§7.2)
→ virtual devices reconnect with the same certificates, same identities, same ACLs;
telemetry resumes with a gap record covering the restore window. Export and import are
audit-logged. Scheduled backup: daily by default with retention pruning; a simulated
unwritable target raises the OPERATIONAL "backup failed" alert. Offsite target: OQ-9, not
a Phase 1 gate.

---

## 6. Alerts v1

**Rule model (AC-ALR-01a, brief 3.4-a):** `metric × comparator × threshold × duration ×
hysteresis × scope × severity`; Phase 1 compares telemetry to the rule's threshold (no
desired state). Evaluation is a persisted state machine per rule × scope: breach shorter
than `duration_s` raises nothing; longer raises **once**; clears after the condition is
absent for the clear window with hysteresis; a restart mid-condition does not re-raise.
Lifecycle `raise → notify → acknowledge → resolve` with actor/timestamp; no "dismiss" verb
exists; no snooze. SAFETY: ack requires an `action_code` (server-rejected otherwise);
resolve refused while the condition is present; banner on every route; channels additive.

**Seeded rules — every physical threshold verbatim from `safety-rules.json` (key path +
`source` string stored on the rule); the comparator is recorded alongside
`source_citation` in the seed and must match the source's wording ("exceeds" → `>`,
"above" → `>`, a band → outside `[min, max]`):**

| Rule | Value (verbatim) | Key path | Tier | Notes |
|---|---|---|---|---|
| Rack air T out of band | 20–24 °C | `climate.air_temperature_c` | ADVISORY | observational |
| Rack RH out of band | 55–70 % | `climate.relative_humidity_pct` | ADVISORY | deadband from `hysteresis_pct: 5`, labelled "borrowed from control"; LLD verifies the night 71–73 % profile actually clears |
| Computed VPD out of band | 0.6–1.0 kPa | `climate.vpd_kpa` | ADVISORY | rule text: derived and tighter than T/RH; may raise while both inputs are in band — correct behaviour (**[HLD confirm → adopted]**, §5.5) |
| Room CO2 enrichment out of band | 1 000–1 500 µmol/mol | `climate.co2_enrichment_target_umol_mol` | ADVISORY, **`enabled: false`** | visible in Rules with enable text "when CO2 enrichment is commissioned (R2)" (**[HLD confirm → adopted]**) |
| Solution temperature high | **`> 26 °C`** ("above 26C") | `irrigation_fertigation.solution_temperature_c.operating_max` | OPERATIONAL | alarm + G3 hold is R2 process; observe only |
| Solution temperature hard inhibit | **`> 28 °C`** ("exceeds 28C"; contracts `interlocks.md` `> 28 °C`) | `…solution_temperature_c.hard_inhibit_c` | **SAFETY (interlock mirror)** | comparator `>`, not `≥`: a mirror must not claim the hardware has inhibited at exactly 28.0 °C when it has not (security F-5) |
| Nutrient pH out of target | 5.6–6.2 | `irrigation_fertigation.ph_operating_target` | ADVISORY | AT-04 and TK-01 pH |
| CO2 safety alarm (mirror) | 5 000 ppm (`≥`, "alarm at 5000 ppm") | `climate.co2_safety_alarm_ppm` | **SAFETY (interlock mirror)** | on the `role: safety-mirror` channel only; rule text: mirrors the independent monitor at 300 mm with a fail-closed solenoid, **is not the safeguard**; detail quotes `co2_asphyxiation_context` verbatim |
| Leak puck tripped / `LSH-04` tripped / E-stop or floor-flood (if advertised) | boolean state | contracts `safety/interlocks.md` hard-wired table | **SAFETY (interlock mirror)** | existence alerts |
| Fan stopped | `fan_tacho_hz == 0` | contracts instrument-tags "the tacho alarm is not optional"; Rack A §05 | OPERATIONAL | a state, not a band; "tacho low" is **[unknown]** and does not exist |
| Stale telemetry / device offline / telemetry gap / broker disconnected / backup failed | design parameters | "design parameter, HLD §10.2" | OPERATIONAL | FR-ALR-06 honesty: no `safety-rules.json` key exists |

**Every hard-wired-interlock mirror is SAFETY tier** (**[HLD confirm → adopted]**): the
interlock has already acted; the alert is the mirror; the console shows "What the hardware
has done" and never a button that suggests it acts on hardware. SAFETY mirror rules have
`duration_s = 0` (a tripped interlock is not debounced by software). **Which samples may
raise a SAFETY mirror (brief §10.1 clarification; security F-7):**

- `good` samples, and `suspect` samples whose reason is a *measurement* doubt (`stuck`,
  `clock_anomaly`, cross-check divergence) — conservative on raise.
- **Never** `implausible-rejected` samples (which is exactly why §4.3's bound invariant
  exists), and **never** `suspect` samples whose reason is a *trust* doubt:
  `topic_mismatch` (the payload's `device_id` disagrees with the topic — §3.1 "never
  trusted"), `unit_unexpected` (a value in an unexpected unit cannot be compared to a
  threshold in the rule's unit; the backend never converts), or a sample from an
  `unclaimed` device. Those raise a security event and/or a device-health OPERATIONAL
  alert instead, never a SAFETY row.

**Resolve bias is the opposite of raise bias (security F-7):** a SAFETY / interlock-mirror
alert **never auto-resolves on the absence of data**. It resolves only on a fresh sample
that is admitted by the list above and is in-band (or, for booleans, un-tripped) for the
clear window, or by an operator resolve when such a sample exists. If the mirrored channel
goes STALE or its device OFFLINE while the alert is raised, the alert stays raised and
surfaces "monitor channel silent since …"; resolve remains refused. Both behaviours are
named unit tests on the alert state machine (LLD).

An `implausible-rejected` sample can therefore never raise or clear a SAFETY mirror.
Enrichment-NDIR ≥ 5 000 while the safety NDIR is silent is a **device-health divergence**
(cross-check), not a second SAFETY rule; Phase 1 records the divergence event without a
tolerance-based rule (tolerance **[unknown]**).

**Not seeded** (brief §5.2): EC band (`TBD`; EC streams with instrument range only, no
badge), G1/G2 gates, DLI, tacho-low, solution-T low bound, CO2 ambient floor, room-vs-rack
divergence, rate-of-change, every fail-safe/electrical entry, `saffron_corm_protocol.*`,
aquatic values. Operator overrides (per zone, non-SAFETY) are labelled "operator-configured,
unsourced". Noisy-rule stats (`acks_without_action_30d`, `auto_resolved_under_5min_30d`)
tag rules `NOISY` above the §10.2 threshold.

**Notification (FR-ALR-04, D-8):** console always; **email via SMTP** (credentials from
compose secrets; a test mail sink in the sim/CI overlay proves delivery, AC-ALR-04a); push
deferred. Per-severity channel configuration lives in **Settings → Notifications**
(not a sixth screen); SAFETY channels can be added, never removed.

---

## 7. Security and authz shape (for `security-safety-reviewer`)

### 7.1 Users, sessions, RBAC, audit

- **Authn:** email + password (argon2id) + **TOTP mandatory** (enrolled on first login via
  admin-issued one-time enrolment); login error never says which factor failed; rate-limited.
  **Sessions:** server-side, opaque cookie (`HttpOnly`, `Secure`, `SameSite=Strict`), idle
  and absolute expiry (values §10.2), revocable; no anonymous route; no "remember device".
- **Roles** exactly per security-architecture §2: `admin`, `grower`, `operator`,
  `technician`, `viewer`. Phase 1 permission matrix, **enforced server-side, mirrored in the
  UI only to hide affordances**:

| Action | admin | grower | operator | technician | viewer |
|---|---|---|---|---|---|
| Read everything (org-scoped) | ✓ | ✓ | ✓ | ✓ | ✓ |
| Alert ack / resolve (SAFETY ack needs action code) | ✓ | ✓ | ✓ | — | — |
| Zone active/idle with reason | ✓ | ✓ | ✓ | — | — |
| Device claim / verify / assign / identify / activate / retire | ✓ | — | — | ✓ | — |
| Non-SAFETY per-zone rule overrides (API only in v0) | ✓ | ✓ | — | — | — |
| Facility metadata (FR-FAC-05, `commissioning_ppfd`) | ✓ | — | — | — | — |
| Users, roles, TOTP reset, notification channels | ✓ | — | — | — | — |
| Backup export, import (CLI), schedule | ✓ | — | — | — | — |

- Org is resolved from the session on every request; no endpoint accepts a client org id.
- **Audit** append-only (§5), covering authn events, alert ack/resolve with reasons,
  backup/export/import, user/role changes, device provisioning/claim/activate, config edits,
  zone status changes, rule overrides, command issuance + ack result.
- Console served by the backend **over TLS** (certificate from compose secrets; LAN only;
  no reverse proxy in Phase 1), standard hardening (CSP, frame-ancestors none, HSTS on the
  LAN host), PWA caches the shell only, **no browser MQTT** (ADR-0009).

### 7.2 Devices and the broker

- **Device credential scheme: mTLS with ECC P-256 client certificates from an offline
  facility CA, for every MQTT-client device class (ADR-0011).** Certificate CN =
  `device_id`; Mosquitto `require_certificate true`, `use_identity_as_username true`;
  ACLs keyed by CN. Rack-controller and I/O-node classes hold **no credential**
  (`credential_type: none`, bus nodes behind the room controller). `credential_type` is a
  per-device-class registry attribute.
- **Principals and ACL prefixes:**

| Principal | Credential | Publish | Subscribe |
|---|---|---|---|
| Room controller `rmc-NN` (virtual in Phase 1) | per-device cert | `trophic/<org>/<site>/<room>/#` | `…/<room>/+/+/cmd` |
| Backend ingest (service) | service cert from the same CA | `…/+/+/cmd` **only** (no `state/desired` in Phase 1 — narrower than IoT §1.1) | `trophic/<org>/<site>/#` |
| Bench-direct ESP32 (only if bring-up needs it; never production) | bench cert, bench CA | `…/<zone>/<device>/#` | `…/<zone>/<device>/cmd` |
| Harness test client (CI/sim overlay only) | sim CA cert | `…/+/+/cmd` (for REJECT tests) | `…/#` | 

  The harness principal exists only in the sim overlay's ACL file, never in the prod ACL.
- **ACL materialisation:** ACLs are derived from the registry (`device_id → acl_prefix`)
  and written by the backend into the broker's ACL configuration at provisioning/revoke,
  followed by a broker reload; mechanism (generated `aclfile` + SIGHUP vs Mosquitto
  dynamic-security plugin) is LLD. The backend is thus in the ACL path only at
  provisioning time, never on the data path. After a restore the backend re-materialises
  ACLs from the restored registry (§5.1).
- **Enrolment channel (Phase 1):** certificate issuance is an operator CLI step against the
  offline CA (`ops/pki/`, key never on the facility host or in the backend container —
  security §9 "offline CA at commissioning time is acceptable"); the registry records
  `credential_fingerprint` and `acl_prefix`; rotation = re-issue same CN, revoke old serial
  (CRL file reloaded). In CI the sim overlay generates an ephemeral CA and certs
  (`simctl pki init`). Device credentials on real hardware live in encrypted NVS (security
  §7) — not a Phase 1 deliverable since no Phase 1 MQTT client is an MCU.
- **Unclaimed = untrusted at both layers:** backend publishes nothing to an UNCLAIMED
  device; edge rejects `device_unclaimed`.
- **No actuation path:** the backend's MQTT wrapper exposes a single command publisher with
  a compile-time allow-list `{sys.identify, sys.ping}`; AC-DEV-03e is a static/contract
  test over that allow-list plus the broker ACL for the backend principal. The sim rejects
  every non-`sys.*` command; firmware v0 initialises no actuator output.

### 7.3 Secrets

Compose secrets files under `.secrets/` — **covered by `.gitignore` as of this branch**
(`.secrets/` was added at the HLD gate; security F-4) — never in the repo or image:
`db_password`, `broker_server_cert`, `broker_server_key`, `broker_ca_cert` (public, but
kept with the set), `broker_crl`, `backend_client_cert`, `backend_client_key`,
`backend_tls_cert`, `backend_tls_key`, `session_key` (session signing + TOTP-secret
encryption), `backup_key`, `smtp_password`. The **CA private key is not a compose secret**
(offline). CI runs a secret scanner (gitleaks or equivalent) as a required job. Rotation of
every item above needs no code change.

### 7.4 Items flagged for the security pass (§14 lists them)

Two-party commissioning fingerprint unattested for rack-link nodes (R2); bench-direct and
harness principals confined to overlays; `override` switch never via `cmd`; backend cmd
allow-list; ACL wording restatement; restore CLI creates no accounts (restores them);
`backup_key` handling in the drill; console TLS; session/TOTP storage; login rate limiting;
SAFETY duration 0 and evaluation on non-rejected samples only; §5.4 relay GPIOs compiled
out (settled, conservative).

---

## 8. Deploy shape and CI

### 8.1 `docker-compose.prod.yml` (build-deploy-engineer)

| Service | Change |
|---|---|
| `backend` | **Uncomment.** `build: {context: ., dockerfile: backend/Dockerfile}` (multi-stage, §2.1); serves API + console on TLS `8443` (replaces the skeleton's `8080`); `secrets:` per §7.3; `depends_on` mosquitto, timescaledb; volumes `backups`, `mosquitto-acl` (write for materialisation); healthcheck. |
| `frontend` | **Stays absent** — the console is a static bundle served by the backend (ADR-0009); the commented block is removed with a one-line note. |
| `mosquitto` | **Uncomment.** `eclipse-mosquitto:2`; listener `8883` TLS with `require_certificate true`, `use_identity_as_username true`, `cafile`/`certfile`/`keyfile`/`crlfile` from secrets; `acl_file` from the `mosquitto-acl` volume; `persistence true`; `max_queued_messages` ≥ 2 × ring capacity (§10.2); no other listener. |
| `timescaledb` | **Uncomment.** `timescale/timescaledb:latest-pg16` (pin a digest in the LLD), `POSTGRES_PASSWORD_FILE`, volume `timescale-data`; not published on a host port. |
| backup job | **In-process scheduler inside `backend`** (daily by default, retention pruning), writing to the `backups` volume; failure raises the OPERATIONAL alert. No separate cron container (one process, one credential set, alerting is native). |
| volumes | `mosquitto-data`, `mosquitto-acl`, `timescale-data`, `backups` |
| secrets | per §7.3 |

### 8.2 `docker-compose.sim.yml` — overlay, not a profile

The sim runs as a **separate overlay file** (`docker compose -f docker-compose.prod.yml -f
docker-compose.sim.yml up`), never as a `profiles:` entry inside the prod file: the prod
file must not mention `sim/` at all, and an accidental `--profile sim` on the facility host
must be impossible. The overlay adds `virtual-room` (`build: ./sim`, scenario and rack count
by env), an init step for the ephemeral sim PKI + harness ACL entries, and a test mail sink.
It is used by CI (short-form scenarios) and by the 72 h soak on the deploy host or dev box.
This is **not** the Phase 2 per-PR integration stack (`docker-compose.integration.yml`),
which stays unbuilt.

### 8.3 `.github/workflows/build-test-deploy.yml`

| Job | Change |
|---|---|
| `detect` | probe `sim/go.mod` and `firmware/rack-controller/CMakeLists.txt` (current probe is `firmware/CMakeLists.txt`); expose `sim` output |
| `backend` | unchanged shape; adds a Postgres+Timescale service container for repository tests, the **org-scoping suite with its coverage gate**, OpenAPI route-vs-spec test, schema conformance test |
| `frontend` | unchanged shape; adds generated-client drift check (`npm run api:generate` → `git diff --exit-code`), ESLint boundary rule |
| `firmware` | `cd firmware/rack-controller`; `idf.py build`; host-side `rack-link/0` conformance test |
| **`sim`** (new) | `go build/vet/test` in `sim/`; conformance test; scenario unit tests |
| **`schema`** (new) | compiles every `docs/iot/schema/v1/*.schema.json`; validates every golden message against its schema with a standalone validator (no subsystem code); lints `docs/api/openapi.v1.yaml` |
| **`boundary-check`** (new) | §2.1 rule 3 |
| **`secrets-scan`** (new) | required |
| **`e2e-sim`** (new) | brings up the sim overlay in CI; runs: short-form soak (minutes), `reject-path`, `commissioning`, `safety-override`, `export-restore` (clean second stack), `home-5s` timing on the 11-rack fixture with seeded history |
| `deploy` | on push to `main` after all jobs: SSH to the facility host or dev box, `git pull`, `docker compose -f docker-compose.prod.yml build && up -d`, `backend migrate`. **Build on host in Phase 1** — no image registry yet (ADR-0008 consequences). Deploy SSH key and host are GitHub secrets managed by `github-ops-agent`. |

The **72 h soak** is not a CI job: it runs from the overlay on the deploy host or dev box
and produces a report artefact under `docs/pipeline/phase-1-foundation/` for
`qa-e2e-validator`.

---

## 9. Acceptance criteria → components

| Locked validation item | Satisfied by | Proven by |
|---|---|---|
| **72 h soak, zero gaps unaccounted** | `sim/` scenario (§4.6); virtual room controller ring + replay; broker persistent session sizing; ingest dedup + gap detection + cause correlation + `ingest_outages`; continuous aggregates; `platform_health` | soak report: stored `seq` ∪ gap rows covers every device's published range; every scripted switch produced exactly its artefacts; aggregates complete; lag non-monotonic; **zero SAFETY alerts** |
| **Commissioning round-trip** | registry lifecycle machine; wizard (§4.4 table); `sys.identify` publisher with allow-list; virtual room controller cmd router + dedup ring; audit | `e2e-sim: commissioning` with two users; unknown class fixture shown unrecognised; failsafe-mismatch fixture blocked at step 2; `operator` refused at claim; six audit rows |
| **Home < 5 s, every number shows source age** | single `GET /facility/summary` with server-side verdict (`ALL CLEAR` only when zero alerts, zero device issues, room controller seen within window, lag under threshold, last backup OK — else `UNKNOWN`); `last_known_state` table (never a hypertable scan); value envelope + one Vue component; polling; shell-only PWA cache | `e2e-sim: home-5s` on the 11-rack fixture with 72 h history: verdict painted within NFR-PER-02 budget (Home < 2 s, tier < 3 s, p95 [proposed convention — qa confirms], p50 reported); every numeric has age + provenance; SAFETY fixture dominant on all five screens; backend cut → `UNKNOWN`, no cached value rendered as current |
| **Org-scoping suite (ADR-0005)** | org-scoped repository layer; session-resolved org; `org_id` on every table incl. audit/telemetry; topic `<org>/<site>` segments; per-CN ACLs | two-org seed; every repository method asserted under org A returns/mutates zero org-B rows; **coverage gate fails the build if any method lacks the test** (mechanism is LLD); no endpoint accepts an org id; Mosquitto ACL test: principal cannot publish outside its prefix |
| **Export/restore into a clean system** | backup module (bundle §5.1, encrypted); `backend restore` CLI; ACL re-materialisation; runbook `docs/ops/restore-drill.md` | `e2e-sim: export-restore` on a second clean compose stack after the soak: counts = manifest; audit intact + append-only; rollups restored; devices reconnect with same identities/ACLs; gap record covers the restore window; tampered byte and incompatible schema refused with reasons; export and import audited |
| **Security review passes** | §7 in full; ADR-0011; AC-DEV-03e static test; SAFETY semantics §6; plausibility never drops; every seeded threshold verbatim | `05-security-review.md` records no open BLOCK |
| NFR-PER-01 ≥ 50 zones | harness parametrised to ≥13 racks (52 zones) [fixture, unsourced] + scripted bursts | every published `seq` stored or gap-recorded; lag non-monotonic |
| NFR-REL-01/02 shape | backend never a dependency of `sim/`; broker local; no internet needed | `AC-REL-01a`: backend container stopped, virtual devices keep publishing; compose passes e2e with outbound blocked |

---

## 10. Decision register

### 10.1 Every [HLD confirm] item — disposition

| Brief § | Item | Disposition | Reason |
|---|---|---|---|
| §1 | `rack-link/0` as the Phase 1 ESP32 transport (5 points), no new ADR, ADR-0007 must reference it | **Adopted** | Strict instance of every guard electronics required; keeps ADR-0004's choke point real from the first commit; keeps the ESP32 clock-free and credential-free; sunset named as a contracts v0.3.0 dependency (§3.2) |
| §3 | `capabilities[]` = hardware inventory + `commands[]` = firmware-accepted; full Rack A inventory on the virtual standard rack; skeleton advertises what it detects; standard profile only; `airflow.tacho` in Phase 1 | **Adopted** | Three specialists incl. the hardware truth-owner need the inventory; the REJECT test gets *stronger* (`action_not_supported` case); Pro is not to be invented; tacho is sourced telemetry ("alarm not optional"). Recorded as ADR-0010 |
| §5.1 | CO2 enrichment band rule seeded `enabled: false` | **Adopted** | Fires nightly under any realistic no-dosing model (AR-8 noise); threshold stays verbatim and visible |
| §5.1 | Every hard-wired-interlock mirror is SAFETY tier | **Adopted** (+ `duration_s = 0`, non-rejected samples only) | Conservative reading; contracts `interlocks.md` lists > 28 °C in the hard-wired table; the interlock has already acted |
| §5.3 | Bound-vs-threshold seeding invariant | **Adopted** as a seed-time hard error | Without it the recommended 5 200 ppm test injection is `implausible-rejected` and the SAFETY path fails silently |
| §5.4 | Relay GPIOs compiled out; polarity/boot state is an R2 HIL item | **Adopted** | Driving pins of unknown polarity is the one way firmware could energise a relay; hardware, not software, stays in the safety path |
| §5.5 | Three climate bands seeded verbatim as independent ADVISORY rules; VPD rule text explains; sim steady state 22 °C / 65 % | **Adopted** | Tightening any band would be inventing a threshold (FR-ALR-06) |
| D-9 | One OPERATIONAL "device offline" alert per device (over health-state-only) | **Adopted** | Connectivity-axis twin of the in-scope staleness alert; root-cause collapse falls out of the health machine (STALE requires ONLINE) |

D-1 … D-16 are all adopted as written; where this HLD adds specificity: D-1 (§2), D-2
(§3.1, `profile_sha256` in the advert), D-3 (§5 seed file + `backend seed`), D-4 (§4.1 DLI),
D-5 (§4.7), D-6 (§5.1), D-7/D-8 (§3.4, §6), D-10 (§3.3 I/O-node profiles), D-11/D-12/D-13
(§4.1, §4.5), D-14/D-15 (console renders only what is advertised; no Desired column), D-16
(`sys.heap_free` on the captured skeleton profile).

### 10.2 Every [proposed default, unsourced] value carried by this design

None of these is a physical or process safety threshold; none may be cited as one; alerts
that use them cite "design parameter, HLD §10.2".

| Parameter | Value | Origin |
|---|---|---|
| Telemetry `interval_s` (all Phase 1 classes) | 30 s, advertised per capability | brief §4 |
| Heartbeat | 30 s | brief §4 |
| `offline_after_s` | 90 s (3 missed heartbeats) | brief §4 |
| `link_timeout_s` | 10 s | brief §4 |
| Staleness `max_age` | `max(3 × interval_s, floor)`; floors T/RH 120 s, CO2 300 s, switches/state reports 120 s, **water-side classes (TE-01/02, LT-02, pH, EC) 120 s — this HLD's floor** | brief §4; HLD |
| Stuck-value N | 20 identical samples → `suspect` | brief §4 |
| `sys.identify.max_duration_s` | 60 s, enforced at the device | brief §4 |
| `ack_timeout_s` / `max_retries` / `ttl_s` / `backend_wait_s` / dedup ring | 5 s / 2 / 60 s / 30 s / 32 — the defaults R2 actuator classes inherit unless overridden | brief §4 |
| Virtual room controller outbound ring | 6 h at nominal rate (~27 k msgs), file-backed | brief §4 |
| Broker `max_queued_messages` (backend session) | ≥ 2 × ring capacity | brief §4 |
| Air-T plausibility (virtual profile) | −40..85 °C, labelled *project-doc placeholder* | device-control-model §6 example |
| RH plausibility | 0–100 % (physics); 95–100 % is `good` | brief §4 |
| Enrichment-NDIR range | 400–5 000 ppm (Phase D SCD41 class) | brief §4 |
| Safety-NDIR | range omitted → class fallback 0–10 000 ppm, recorded as fallback | brief §4/§5.3 |
| pH / EC ranges | 0–14 / 0–5 mS/cm open-ended upper | brief §4 |
| **Water-temperature class fallback** | **0–100 °C, labelled "liquid-water physics bound, class fallback — not an instrument range"**; exceeds 28 °C per §4.3 | HLD (brief asked for it); the part range stays **[unknown]** |
| Sim steady state | 22 °C / 65 % RH | brief §4 |
| Alert `duration_s` for band rules | LLD sets per rule, labelled `unsourced-default`; structural constraint: ≥ 2 × the metric's `interval_s` so one late/suspect sample cannot raise; SAFETY mirrors `0` | brief §5.2; HLD |
| NOISY tag threshold | `acks_without_action_30d ≥ 10` or `auto_resolved_under_5min_30d ≥ 20` | HLD (UX §6.3 asked for it) |
| Export bundle raw-telemetry window | 30 days by default, manifest-declared, ops-configurable | HLD (AC-BKP-01b) |
| Scheduled backup | daily 02:00 facility-local, retention 14 bundles | product AC-BKP-03a / UX §3.6 |
| Session expiry | idle 12 h, absolute 7 d (LLD may tighten) | HLD |
| Load fixture | 13 racks × 4 tiers = 52 zones | brief §8.3 |
| NFR-PER-02 percentile | p95 (p50 reported) | product [proposed convention — qa confirms] |
| Console polling interval / backoff | LLD | UX §7 |

### 10.3 Every [unknown] — left unknown

Air-T sensor part and range (Q-3, H3) · water-temperature element part and range (Q-3) ·
CO2 safety monitor part and range, must exceed 5 000 ppm (Q-2) · fan model, pulses/rev,
"failed/low fan" band (Q-3, H4) · EC operating target (`TBD`; Q-7) · G1 gate reconciliation
(contracts 1.5× vs OCR ~1.2×; maintainer loop) · DLI target (`TBD`) · alert duration
windows (LLD defaults, labelled) · solution-temperature low bound · CO2 ambient floor ·
room-vs-rack T/RH divergence tolerance (Q-8) · per-channel rate-of-change limits ·
leak/`LSH-04` wiring (contact-in-coil vs GPIO; Q-4, R2 blocking) · relay board polarity and
strapping pins; fan and LED de-energised states (Q-5, H6/H7) · Modbus register map (Q-6,
H10) · ESP32 variant and framework (H2; board config) · first Ooty racks GROW vs GROW-S
(Q-1) · whether the room I/O node reads E-stop/floor-flood state · leak/LSH-04 debounce
time (LLD, not a safety threshold) · offsite backup target (OQ-9) · whether a board LED is
visible through the enclosure. None blocks Phase 1; each blocks the R2+ feature that would
consume it.

**For LLD (carried from `05-security-review.md`, not unknowns — design work the LLD must
show):**

- **F-2 — sim overlay trust anchor and ACL persistence.** The overlay's broker trust is the
  sim CA **only** (it replaces, never appends to, `broker_ca_cert`), so no facility-CA
  principal can connect while the overlay is up and vice versa; harness ACL entries live in
  an overlay-only ACL file, or the backend's materialiser regenerates the whole ACL file
  from the registry on every start and drops anything not derived from it; the runbook
  states that after any overlay run on the facility host the broker is restarted with prod
  secrets and the ACL re-materialised; the e2e asserts a sim-CA certificate is refused by
  the prod-configured broker.
- **F-3 (with F-12) — bundle scope and key travel.** `sessions` (live bearer ids) and any
  other ephemeral table are **excluded** from the bundle and declared as such in the
  manifest; TOTP secrets are encrypted under a dedicated `totp_key` split from
  `session_key` (different rotation semantics), and `docs/ops/restore-drill.md` states that
  the operator's secret set carried to the clean stack includes `totp_key` and
  `backup_key`; the `export-restore` e2e logs in with TOTP after restore, not just counts
  rows. §5 Identity row and §7.3 are read with this amendment.

---

## 11. Tradeoffs considered and rejected

| Considered | Rejected because |
|---|---|
| Sim as a virtual build target under `firmware/` | Would make virtual devices a firmware product and invite shared source across the firmware/backend line; profiles are data, the captured advert is a diff target, and `sim/` as its own Go module keeps a later firmware/hardware extraction a directory move |
| Sim as a `backend/` package | Violates ADR-0008 rule 2 by construction and makes "stopping the backend stops nothing in the room" untestable |
| Virtual racks publishing MQTT directly "for convenience" | Would leak into the design as a second field client type (ADR-0004 point 1) and pre-empt OQ-3 (electronics Q5.5) |
| Direct-MQTT-from-ESP32 as the Phase 1 transport | Persisting it creates the client type ADR-0004 forbids; the ESP32 has no NTP path and would need a TLS stack it will not ship with |
| Pruned advert (sensing + `sys.*` only, IoT §0.4) | Registry would be untruthful about the rack for R2, UX step 2 failsafe cross-check impossible, and the REJECT test loses its stronger `action_not_supported` case (ADR-0010) |
| Separate "hardware profile" and "firmware profile" documents | Two artefacts to keep in sync for one device; the two-field profile carries both truths in one advert |
| Per-device PSK (security §2 "first-phase compromise") | The MCU constraint that justifies it applies to no Phase 1 MQTT client; Mosquitto cannot mix PSK and certificate auth on one listener, so PSK would add a listener and credential path to be removed later (ADR-0011) |
| Code-first OpenAPI (generate the spec from Go) | The spec would follow the implementation rather than bind it; spec-first lets both sides generate and CI check drift symmetrically |
| Separate `frontend` container (nginx) | ADR-0009 fixes the console as a static bundle served by the Go backend; a second web server is a second TLS endpoint to harden |
| Compose `profiles: [sim]` inside `docker-compose.prod.yml` | The prod file must not reference `sim/`; an overlay makes the sim's absence on the facility host structural |
| Backup as a separate cron container | Second DB credential, second image, and the failure alert would need a side channel into the alerts module |
| Console-driven restore | A clean stack has no accounts to log in with until the bundle restores them; would need a bootstrap-admin path (D-6) |
| Topic-root schema versioning (`trophic/v1/…`) | Consumers must see the version before parsing; `v` in the payload does that and keeps ACLs stable |
| Telemetry batching (one message per frame) | Phase 1 rate (~3 msg/s facility-wide) does not need it; per-sample messages keep ACL and gap detection simple; R2 optimisation if ever |
| Pull backfill (`sys.backfill`) in Phase 1 | Needs the real room controller's local store (OQ-3/ADR-0007); push-on-reconnect from the ring is enough for "zero gaps unaccounted" |
| SSE/WebSocket live updates for the console | Polling meets Phase 1 load (ADR-0009); a backend-owned relay is a later `/extend` if polling proves insufficient |
| Row-level security in Postgres now | Deferred per ADR-0005; the repository convention + coverage-gated tests is the R1 discipline |
| Pro rack profile in the soak (botany §4.1) | Not to be invented until the hardware program publishes it; the unknown-class fixture already exercises optional-capability rendering |
| Per-PR integration compose stack | Explicitly Phase 2 CI/CD (`docs/pipeline/README.md`) |
| A "tacho low" ADVISORY row (UX wireframe) | The band is **[unknown]**; only `fan_tacho_hz == 0` is sourced |

---

## 12. ADRs opened by this HLD — and why

| ADR | Status | Why an ADR (outlives this requirement) |
|---|---|---|
| **ADR-0010 — Capability advert semantics: hardware inventory vs firmware-accepted commands; `sys.*` controller command namespace** | proposed (amended at the HLD gate for F-1) | Refines what "advertise" means in ADR-0002 for firmware, sim, registry, console and edge alike; registers `sys.*` as a closed set of `control.compute` actions in the device-control-model §3 vocabulary with an admission rule; inherited by every release, otherwise re-litigated in R2 |
| **ADR-0011 — Device credential scheme, per device class** | proposed | security-architecture §2 requires it "in an ADR when firmware work starts"; §6.6 security pass lists it; decides mTLS (ECC P-256, offline CA) over PSK with the reasoning in §7.2 |

**Not opened:** the `rack-link/0` transport (application of ADR-0004 point 1; sunset to be
folded into ADR-0007's consequences), the topic/schema refinements (amendment to
device-control-model §4 + `docs/iot/schema/v1/`), `sim/` placement (application of ADR-0008
as a fourth subsystem, nothing split), and every D-item in the brief (release-scoped design
choices, not architecture that future work must re-derive). `docs/adr/README.md` has no
index table, so none was added.

---

## 13. Named Phase 1 outputs beyond code

| Output | Owner | When |
|---|---|---|
| **Schema v1 hardware-side review request** — `docs/iot/schema/v1/` (JSON Schema per message type + profile schema + golden messages + `rack-link-0.md`) submitted through the trophic-contracts CHANGELOG process (instrument-tags "hardware must review"). Agenda: units gaps `degC`, `pct`, `Hz`, `bool` → `units.md`; a tag for the rack T/RH node; per-rack `LSH-04` tagging; safety-NDIR part and range (must exceed 5 000 ppm); air-T and water-T part ranges; fan pulses/rev; Modbus register map as the **v0.3.0 dependency that retires `rack-link/0`**; result recorded by bumping the instrument-tags row to "v1 reviewed <date>" | hld-architect drafts the request text; `github-ops-agent` files it | at ship |
| **`docs/iot/device-control-model.md` §4 amendment** — explicit `<room>` segment; `<tier>` → payload `scope`; retained `capability` leaf; `event` type list; envelope fields; §6 example staleness values promoted to labelled design parameters; RK-A-ROOM Rev 3 → Rev 4 citation | `docs-writer` | at ship |
| **CLAUDE.md boundary list gains `sim/`** (ADR-0008 rules 1–4 apply to four subsystems; no supersession) | `docs-writer` | at ship |
| ADR-0007 (OQ-3), when drafted, **must reference `rack-link/0`** and its sunset | whoever drafts ADR-0007 | OQ-3 close |
| `requirements.md` annotation "FR-SIM-02 partially delivered R1 (offline/stale/implausible + test-only override)"; `safety-rules.json` G1 reconciliation (contracts 1.5×) | maintainer loop | next `/health-check` |
| `docs/ops/restore-drill.md` runbook (the committed drill the e2e follows) | build-deploy-engineer / docs-writer | build |
| `docs/ux/wireframes/` low-fi pass for the five screens before LLD (AC-DEL-02); IA §6 review scheduling (Q-9) | owner / ux-product-designer | before step 6 |
| 72 h soak report artefact under `docs/pipeline/phase-1-foundation/` | qa-e2e-validator | acceptance |
| Relay-polarity bench check recorded as the **R2 bring-up acceptance item** gating the first firmware version to map relay pins; "de-energised as first boot action" recorded as that version's requirement | firmware-engineer (R2) | R2 |

---

## 14. Flags for downstream steps

**`security-safety-reviewer` (step 05):** §7 in full; §7.4 list; the ADR-0011 choice; the
SAFETY-tier rule (`duration_s = 0`, non-rejected samples only) and the §4.3 seeding
invariant; §5.4 relay GPIOs compiled out (settled, conservative); the harness test client
principal existing only in the sim overlay; `override` never via the device `cmd` schema;
AC-DEV-03e allow-list; the ACL wording restatement; two-party commissioning fingerprint
unattested over rack-link (R2 item); backup encryption key handling in the drill.

**`lld-architect` (step 06):** OpenAPI v1 with the value envelope and `as_of`; DDL for §5;
the repository org-scoping convention and the coverage-gate mechanism; ACL materialisation
mechanism; band-rule `duration_s` defaults (labelled `unsourced-default`); RH hysteresis
semantics checked against the night profile; VPD pairing window; chart raw/1 m/1 h cut-over;
polling interval; bundle encryption tooling; `simctl` control surface (local CLI/HTTP);
`rack-link/0` frame grammar; ESP-IDF component layout with board config (variant
**[unknown]**); F-10 CEA/aquarium wording check.

**`firmware-engineer`:** advert from detected hardware; status-LED-only identify; JSON-lines
`rack-link/0` over UART with `link_seq` + `t_mono_ms`; register-image sample path; **no
MQTT/TLS/SNTP; no relay GPIO initialised, mapped, or named**; optional `sys.heap_free`; the
72 h run on real hardware if any is available (heap stability). **`backend-engineer`:** §4.1
step table verbatim; staleness on `ts_ingest`; backfilled rows never touch live state or
alerts; single command publisher with allow-list. **`frontend-engineer`:** UX §4 value
envelope component; SAFETY banner in the shell; no Desired column; renders only what is
advertised; SIM tags; shell-only PWA cache. **`build-deploy-engineer`:** §8.
**`qa-e2e-validator`:** §9 table is the acceptance map; Q-1 (GROW vs GROW-S) changes only
the optional "≥1 real rack" shape.
