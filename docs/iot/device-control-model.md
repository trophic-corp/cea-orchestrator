# Device & control model — desired, command, state, telemetry, faults

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** device/control model of the foundation analysis. Hardware status
below is sourced from the Rack A engineering set (RK-A-SYS Rev 2, RK-A-MFG Rev 2,
RK-A-WRS Rev 3, RK-A-ROOM Rev 3 — the current revision chain) and Phase D BOM; nothing
in §2 is inferred. OCR-unreliable values are treated as unknown (see audit).

## 1. The five-way separation (normative)

Every controlled thing is modeled as five distinct records, never conflated:

```
1. DesiredState      what the operator/recipe asked for        (authoritative at edge controller)
2. Command           the message sent to hardware              (edge controller → device, idempotent)
3. Acknowledgement  device's verdict on the command            (SUCCESS | REJECTED(reason))
4. ReportedState     what the device says it is doing now      (LED = 69.7%)
5. Telemetry         measured values + events                 (sensor stream, quality-flagged)
```

The brief's canonical example (Desired 70% → `SET_LED(70)` → `SUCCESS` → reported 69.7%)
is exactly this pipeline, and each stage is separately queryable in the UI and stored with
history. **Alerts/faults** are a sixth, derived layer (rules over 1–5), never a substitute
for any of the five.

## 2. Hardware capability status (evidence-based)

Legend: **SUPPORTED** = in the current rack/room build being manufactured; **PLANNED** =
named with a timeline in the engineering docs; **OPTIONAL** = documented option at extra
cost (Pro/R&D build or sourcing option); **ABSENT** = not in the design (several
explicitly rejected by the engineering docs — those rejections are design decisions the
software must respect, not gaps to fill).

| Capability (per tier unless noted) | Status | Evidence (engineering docs) |
|---|---|---|
| Flood-and-drain scheduling (tier fill/dwell/drain; NC fill + NO drain solenoid per tier) | **SUPPORTED** | 4 fill + 4 drain solenoids on 8-ch relay; ESP32 rack controller "runs the tier sequence locally"; flood 22 mm / dwell 10 min; drain rate 4.2× fill |
| LED dimming (0–10 V pair per tier; 2 bars share one pair) | **SUPPORTED** (infrastructure; fixtures procured separately) | "48 V DC + 0-10 V dimming pair per tier, IP65 keyed connector"; drivers remote in end enclosure |
| LED photoperiod switching | **SUPPORTED (mechanism OQ)** — dark period exists ("dehumidifier… for the dark period") but on/off actuation (dim-to-off vs relay) depends on the not-yet-chosen LED driver model → [OQ-14] |
| Airflow control (PWM per tier) + tacho feedback | **SUPPORTED** | EC fan per tier, 178 m³/h, "PWM or 0-10 V from the rack controller, with tacho feedback"; tacho alarm "not optional" |
| Ducted plenum airflow (CV 33% → ~10%) | **PLANNED** (Phase 2 retrofit, "month 2-3", ₹2,640/rack, bolts to same grid) | same fans thereafter, distributed discharge |
| Air T/RH sensing | **PARTIAL** — rack-level (1 per rack @ tier 3) standard; **per-tier = OPTIONAL** (Pro/R&D build "+per-tier sensing") | "Room HVAC owns bulk temperature, humidity and CO2" |
| Temperature/humidity *control* per tier | **ABSENT** (room-level by design) | HVAC 2.0 TR inverter + 50 L/day dehumidifier = room scope |
| CO2 enrichment (PID) + CO2 safety | **SUPPORTED** (room-level) | NDIR @ 1.5 m (control) + NDIR @ 300 mm (safety); fail-closed safety solenoid; alarm 5000 ppm; cylinder on terrace |
| Per-rack / per-tier CO2 sensing | **ABSENT** (explicitly rejected) | "…measures the same room sixteen times and costs sixteen times as much" |
| Water temperature sensing + interlock | **SUPPORTED** | TE-01 in TK-01 (terrace), TE-02 in trough; alarm 26 °C, MV-01 inhibit ≥ 28 °C |
| pH/EC sensing | **SUPPORTED** — room-shared (TK-01 + trough), NOT per rack/tier | trough AT-03/AT-04 60-s rolling means per recovery batch; dosing proportional to EC deficit |
| Leak detection | **SUPPORTED** — per rack (puck @ drain header, hard-wired to rack controller, bus-independent) + room floor flood (hard-wired to MV-01) | 60 s solenoid-closure proof is an existing QC hold point |
| Water level | **SUPPORTED** — system level only (TK-01 LT-02, trough floats ×3, header LSH-04, make-up mechanical float); **per-tray ABSENT by design** | "the overflow standpipe sets level mechanically, which deletes sixteen sensors per rack" |
| Flow metering | **SUPPORTED** (down-main totaliser; 1.3× over-delivery → close MV-01 + alarm; FS-01 transfer proof) | |
| Fertigation dosing (A+B+acid peristaltic, EC-proportional) | **SUPPORTED** (DOS-01, room level) | injects into rising main |
| UV disinfection + filtration + 8-gate reuse logic | **SUPPORTED** (water skid) | G1–G8 gate table; UV ≥100 mJ/cm² design dose, G5 = ≥70% rated intensity |
| HVAC / dehumidification | **PLANNED** (specified: 2.0 TR 3-ph inverter + 50 L/day deh.; circuits C12/C13 allocated; not yet installed) | |
| Master/diverter valves (MV-01 spring-return closed; FV-01 spring-return to waste) | **SUPPORTED** | |
| PAR/PPFD sensing | **PORTABLE only** — quantum sensor on jig at commissioning/quarterly; no fixed PAR sensor | |
| IR leaf temperature (true VPD) | **OPTIONAL** (Pro only) | |
| Imaging / phenotyping | **OPTIONAL/FUTURE** (Phase D sourcing entry only; nothing installed) | |
| Per-rack electricity metering | **ABSENT** (per-rack circuits exist; no kWh meter documented) → energy per rack starts as **calculated estimate** (schedule-derived) | |
| Room HVAC integration with platform setpoints | **PLANNED** (hardware specified; control integration is platform Release 7 scope) | |
| Controller compute | **SUPPORTED**: 11× rack controllers (ESP32 + 8-ch relay + RS485), room controller (bus master, local TSDB, local UI, UPS), terrace I/O node, water-skid node, room I/O node — "14 bus nodes" | |
| In-room bus | **SUPPORTED**: Modbus RTU over RS485, multi-drop, 120 Ω both ends, wired-first ("a room of galvanised racks is a poor 2.4 GHz environment") | |

**Documented control allocation** (the platform must slot into this, not reinvent it):

> Rack controllers own the tier fill/dwell/drain sequence, fan PWM, and the leak/header
> interlocks, and hold a last-known-good schedule. The room controller owns the drain
> token, the eight-condition reuse gate, dosing, HVAC and CO2 setpoints, alarms, and the
> local log. The cloud owns remote view and history — and **nothing the room depends on**.

Local interlocks (leak puck, LSH-04, E-stop, MV-01 flood sensor) are **hard-wired and
bus-independent**; software reads their state but is not in their safety path.

## 3. Capability classes (vocabulary)

Versioned registry (ADR-0002); each class has parameters, units, and command semantics:

```
light.dim            {min,max}          per-tier 0-10 V            (directive at tier scope)
light.schedule       {on,off}           per-tier photoperiod       (directive at tier scope)
light.state_report   —                  actual intensity/on-off
airflow.speed        {min,max,pwm}      per-tier fan + tacho       (directive at tier scope)
airflow.tacho        —                  fan health (SUPPORTED telemetry, alarm-mandatory)
irrigation.flood     {depth_mm,dwell_min,cycle}  per-tier schedule (directive at tier scope)
valve.control        {failsafe_state}   fill/drain/master/diverter (guarded; fail-safe states law)
climate.heat / climate.cool / climate.humidify / climate.dehumidify   (room scope)
co2.enrich           {target,alarm}     room scope, interlock-locked (fail-closed safety solenoid)
dosing.fertigation   {profile}          room scope
sense.temp_rh / sense.co2 / sense.ph / sense.ec / sense.water_temp
sense.water_level / sense.flow / sense.leak / sense.par / sense.smoke
meter.water / meter.power               (absent now → estimates, provenance-marked)
control.compute      —                  controller role (rack | room | io-node)
```

**Scope directive:** when a recipe is applied to a tier zone, `light.*`, `airflow.*`,
`irrigation.*` are **directive** (enforced on that tier). `climate.*` and `co2.*`
parameters on a tier recipe are **advisory at tier scope / directive at room scope**: the
platform records them as the tier's demand, but the actuator lives at room level — the
UI must show room setpoints as *shared*, and conflicting tier-level demands are surfaced
as a warning (recipe wants 22 °C, room runs 20–24 °C band → OK; two tiers demanding
incompatible climate → operator decides). Never pretend tier-level climate control the
hardware doesn't have.

## 4. Transport & topics

Two transport domains (ADR-0004, aligned with the documented hardware):

1. **Field bus (in-room):** Modbus RTU over RS485 — rack controllers, I/O nodes → room
   controller. Not our design to change; the platform's device model maps onto it.
2. **Facility backbone:** MQTT 3.1.1/5 over TLS :8883, broker on the facility host
   (Mosquitto). The **room controller is the edge MQTT client** (bus master + local
   historian); backend services and (later) a cloud/remote reader are separate clients.
   Topic taxonomy, versioned:

```
trophic/<org>/<site>/<room| rack>/<tier?>/<device>/
    telemetry/<metric>     (QoS1, retained=false)
    state/reported         (QoS1, retained=true  — last known actual state)
    state/desired          (QoS1, retained=true  — edge-controller-owned copy)
    cmd                    (QoS≥1, per-command id; ack required)
    ack                    (QoS1 — result + reason + state-after)
    event                  (QoS1 — lifecycle: boot, claim, capability advert, faults)
```

Per-device topic ACLs (security-architecture §3). Retained desired/reported states make
reconnect recovery deterministic.

## 5. Command lifecycle

```
origin: console/operator ──► backend validates (authz, envelope, capability exists)
        ──► publishes config/command to edge controller (MQTT, cmd id + expected state)
edge controller: validates AGAINST (a) device capability profile, (b) safety envelope
        (safety-rules.json bounds), (c) interlock states (E-stop active? leak tripped?
        flood window constraints? tank level?) 
   ├─ REJECT → ack{REJECTED, reason, code}  → surfaced to operator, logged, alertable
   └─ ACCEPT → issue to device (RS485/MQTT), await device ack with timeout
        ├─ device ack SUCCESS → update ReportedState, publish ack up
        ├─ device ack REJECT   → propagate reason (device-side envelope violation)
        ├─ timeout → bounded retry (idempotent cmd id; same id never double-applies),
        │            then ack{TIMEOUT}; device marked command-degraded
        └─ offline device → command parked with TTL; expired commands discarded with
             event (never silently queued forever, never auto-applied late)
```

Rules:
- Commands are **idempotent** (operator double-tap, broker redelivery, edge retry → one
  effect). One-shot commands (extra flood now) carry expiry + "already-applied" semantics.
- Ack is **per-command**, not per-batch-of-changes: partial application states are
  representable (3 tiers applied, tier 3 rejected — why).
- The edge controller is the only thing that ever talks to actuators (NFR: cloud never
  in the actuation path; console shows acks, never assumes).
- **Offline**: schedules continue at the edge from last-known-good desired state. The
  console renders zone data with explicit age + "edge-autonomous" mode banner; commands
  queue only if the *edge* (not the cloud link) is reachable.
- Controller reboot: last-known-good desired state + schedules persist; on boot,
  self-check → resume schedules; one-shot commands are NOT replayed; a reboot event is
  telemetry (alertable) and the UI marks the zone "controller restarted at HH:MM".
- Power restoration: firmware boots to documented de-energized fail-safe states
  (safety-rules.json `valve_and_actuator_fail_safe_states`) before schedules resume —
  the rack controller "keeps irrigating if the bus or room controller drops", and
  hardware fails safe if *it* drops. Commissioning proves it (QC hold point).

## 6. Telemetry quality & plausibility

Every sample carries: `ts_source` (device clock), `seq` (per-device counter),
`quality ∈ {good, suspect, implausible-rejected}`, `calibration_id` (§7).

Gates at ingest (edge + backend):
- **Range plausibility** per sensor class (physical: RH 0–100%, T −40..85 °C class,
  CO2 0–10 000 ppm…): values from the sensor's documented range, not crop targets.
- **Rate-of-change plausibility**: change faster than the sensor can physically move
  → suspect.
- **Stuck-value detection**: N identical samples over the sensor's noise floor → suspect
  (device fault, not plant news).
- **Cross-checks where they exist** (two NDIRs, room vs rack T/RH): divergence → both
  suspect + device-health alert.
- Verdicts flag, never silently drop (FR-TEL-02); consumers filter by quality.
- **Staleness**: per-class max-age (e.g., T/RH 2 min, CO2 5 min); exceeded → zone badge
  + OPERATIONAL alert. Portal/operator views always display data age.

Impossible-value handling (brief's "sensor gives impossible values"): flag → device
health alert → operator workflow (UC-19) offers: recalibrate, replace, or mark
out-of-service. A sensor faulted to a *safe-looking wrong value* (stuck mid-range) is the
hardest case — cross-checks + stuck-detection exist precisely for it; anything acting on
a single sensor's word for a safety decision is a design violation.

## 7. Calibration metadata

- Registry per sensor: model, range, accuracy class (Phase D specs: e.g., CO2 SCD41
  ±50 ppm, T/RH ±0.3 °C/±2%), calibration date + due date, calibration method record.
- Calibration expiry → maintenance alert; **expired-calibration sensors are
  provenance-marked** in telemetry (quality=suspect after expiry) — R&D joins must be
  able to exclude them.
- Reservoir-level pH/EC calibration is room-shared by design ("one calibration for the
  whole room instead of one per rack") — calibration workflow targets TK-01 + trough
  probes explicitly.

## 8. Overrides & emergency states

- **Manual override**: per-actuator, time-boxed (default max duration per class, e.g.,
  light 24 h / flood 4 h), requires confirmation showing envelope impact; banner while
  active; auto-revert to desired state with event; fully audited (FR-CTL-04).
- **E-stop**: hardwired, room-level, per aisle — software *reflects* it. While engaged:
  conflicting commands refused (`REJECTED: e-stop`), platform shows exactly what the
  E-stop dropped (fill solenoids, MV-01, pumps; drains de-energize open), and a
  post-incident review package (events around the window) is one click.
- **Interlock-tripped zone** (leak puck, LSH-04, flood sensor): zone marked inhibited;
  recovery requires explicit operator action + reason; nothing auto-resumes.
- **Fail-safe transparency**: the UI can always answer "what happens to X if power is
  cut right now?" per device class (from `valve_and_actuator_fail_safe_states`) — this
  is education, and it's testable in simulation (FR-SIM-02).

## 9. Fault model (device failure / network loss / invalid commands)

| Failure | Edge behavior | Platform behavior |
|---|---|---|
| Device offline | controller marks node dead; schedules degrade per-class policy (e.g., valve: skip cycle + alert; fan: last PWM held, tacho alarm) | device health alert; zone badge; commands parked w/ TTL |
| Stale telemetry | staleness gates → suspect → alert | age badge everywhere |
| Impossible sensor values | §6 gates | quality flag; device health workflow |
| Pump failure | documented: 4-min run limit catches stuck float; duty/standby auto-changeover (P-02 A/B) | maintenance alert w/ context |
| Insufficient water | LT-02 < 90% gate holds recovery; MV-01 inhibited ≥28 °C; make-up is mechanical float | inventory-of-water visibility + alerts |
| Temperature exceeds safety limits | room controller alarms + HVAC commands per envelope | SAFETY alert (hardware interlocks where applicable); operator workflow |
| Controller reboot | §5 power-restore behavior | reboot event, zone marked, audit |
| Network loss (bus) | racks run locally; room controller logs; "an alarm, not a stoppage" | connectivity status per zone; gap-marked telemetry |
| Invalid command | rejected at controller w/ reason (§5) | surfaced to origin; repeat-rejection alert |
| Firmware update mid-operation | update policy: no updates during flood window or active interlock; dual-partition rollback (security-architecture §6) | fleet view; staged rollout |

## 10. Simulation contract

A simulated device advertises the *same capability profile schema* as real hardware
(FR-SIM-01) — the registry, command pipeline, telemetry pipeline, and UI cannot tell
the difference except by an explicit `sim: true` marker. Fault injection (FR-SIM-02)
drives the §9 table: each row must be reproducible in CI before it can be claimed to
work in the room. The HIL pass in the pipeline (`automation-engineer`) uses the same
contract against real IO.