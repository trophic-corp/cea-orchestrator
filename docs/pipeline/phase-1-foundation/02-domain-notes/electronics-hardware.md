# 02 — Domain notes: electronics & hardware (firmware v0 feasibility)

**Pipeline:** `/ship` · **Slug:** `phase-1-foundation` · **Date:** 2026-09-10
**Author:** electronics-hardware specialist · **Consumes:** trophic-contracts **v0.2.0**
(`sensors/instrument-tags.md`, `capabilities/absent-by-design.md`, `conventions/units.md`,
`electrical/elv-boundary.md`, `safety/interlocks.md`), `safety-rules.json` (2026-09-05),
Rack A Specification §04/§07/§09/§11, Floor Plan §05–§07, Phase D BOM §3–4, ADR-0001/0002/0004,
`docs/iot/device-control-model.md`, OQ-3 decision brief.

**Verdict in one line:** Phase 1 firmware v0 (capability advert + telemetry + no-op ack, no
actuation) is feasible on the documented rack-controller platform with no changes to the
hardware set — **provided** the HLD does not let the ESP32 become the production MQTT/TLS
client (it is a Modbus RTU slave in the documented architecture) and does not invent the
Modbus register map, the sensor part numbers, or the LED driver behaviour, all of which the
contracts mark as unpublished.

Every number below is tagged **[sourced]** (with the document), **[derived]** (arithmetic on
sourced values — not itself a rule), **[general]** (component-class engineering knowledge,
not from the program's sources; verify against the chosen part's datasheet), or **[unknown]**.

---

## Q1. Rack controller platform and what a standard rack actually senses

### 1.1 Platform — confirmed

| Item | Value | Source |
|---|---|---|
| Controller | "ESP32-class with 8-channel relay and RS485 transceiver. Runs the tier sequence locally and holds last-known-good setpoints" | Rack A Spec §07 p.15 **[sourced]**; contracts `instrument-tags.md` **[sourced]** |
| Count | 11 rack controllers, of 14 bus nodes (11 racks + terrace I/O + water-skid + room I/O) | contracts `instrument-tags.md`; Floor Plan §07 **[sourced]** |
| Bus role | **Modbus RTU slave** on RS485 multi-drop, daisy-chain, 120 Ω both ends, room controller as bus master at one physical end; "one address per rack" | Rack A Spec §07 p.14–15; Floor Plan §07 **[sourced]** |
| MQTT/TLS | Belongs to the **room controller** ("Room controller polls the bus, writes to a local time-series database"; "Cloud and app: MQTT over TLS"). The rack controller is **not** an MQTT client in the documented build | Rack A Spec §07 p.13–15; ADR-0004 decision 0–1 **[sourced]** |
| Wireless | "Wireless stays available as a retrofit for a rack that cannot be cabled" — a retrofit exception, not the standard path | Rack A Spec §07 p.15 **[sourced]** |
| Power rail | 24 V DC PSU feeds "fans, solenoids, controller, sensors"; total rack connected load 316 W, 1.37 A at 230 V | Rack A Spec §07 p.14 **[sourced]** |
| Controller's own power draw, PSU rating, relay coil voltage/current | **[unknown]** — not in any source. Not needed for Phase 1 (no actuation). |
| ESP32 variant (classic WROOM-32 / S3 / C3), flash, PSRAM | **[unknown]** — "ESP32-class" only. Phase D §3 lists "ESP32 (Wi-Fi/BLE) or STM32" for sensor nodes generically. **Firmware skeleton must keep the target in a board config, not in code.** |
| Firmware framework (ESP-IDF vs Arduino) | **[unknown]** / not decided. Not a hardware spec; LLD choice. Recommendation only: ESP-IDF, because NVS-encrypted credentials (security-architecture §7) and TLS buffer tuning (Q3) are first-class there. |

### 1.2 Per-rack I/O — what the controller is wired to (standard GROW build)

**Sensed inputs (these, and only these, may appear in the standard virtual profile):**

| Channel | Qty/rack | Interface | Location | Source |
|---|---|---|---|---|
| Air temperature + RH (one node) | **1** | I²C digital ("I²C to the T/RH node … in the dedicated channel") | Front-right upright, vented radiation shield, mid-depth of **tier 3**, 1239 mm | Rack A Spec §04 p.8, §07 p.15, §09 p.18; contracts `absent-by-design.md` ("rack-level, one per rack at tier 3, is standard") **[sourced]** |
| Leak detection puck `LD-01..LD-11` | **1** | Dry contact | Floor spot sensor at the base of the drain header, right rear, 5 mm above floor | Rack A Spec §04 p.8; contracts `instrument-tags.md`, `interlocks.md` **[sourced]** |
| Header high-level switch `LSH-04` | **1** | Float or optical level switch, hard-wired to the rack controller, bus-independent | DN40 drain header at 200 mm, just below tier 1 drain entry | Rack A Spec §11 p.21; contracts `interlocks.md` **[sourced]** |
| Fan tacho | **4** (1 per tier) | Pulse input, "3-core to each plenum end cap" (24 V + PWM + tacho) | Plenum end cap, right-hand end of each tier | Rack A Spec §05 p.11, §07 p.14; contracts `instrument-tags.md` ("the tacho alarm is not optional") **[sourced]** |

**Actuator outputs (present in hardware; advertised as hardware capability; NOT driven by v0):**

| Channel | Qty/rack | Interface | Fail-safe (de-energized) | Source |
|---|---|---|---|---|
| Fill solenoid, NC | 4 | Relay ch (24 V switched) | **CLOSED** | `safety-rules.json` `valve_and_actuator_fail_safe_states` **[sourced]** |
| Drain solenoid, NO | 4 | Relay ch (24 V switched) | **OPEN** | same **[sourced]** |
| Fan speed | 4 | PWM or 0–10 V | Not stated in safety-rules; fans "run continuously, including through the dark period" (Rack A Spec §05 p.11) — **[unknown]** as a formal fail-safe state; do not invent one |
| LED dim | 4 pairs | 0–10 V per tier (2 bars per pair), 48 V DC supply, driver remote in end enclosure | **[unknown]** — LED driver "not chosen"; dim-to-off vs relay is [OQ-14] | contracts `instrument-tags.md` "Still missing" **[sourced]** |

**Explicitly absent from a standard rack — the virtual profile must not advertise these**
(contracts `absent-by-design.md`, Rack A Spec §04 p.9):
per-tier T/RH (Pro/R&D **option** only, 4 per rack), per-rack/per-tier CO₂ (rejected),
per-tray water level (rejected — standpipe collar sets level), fixed PAR/PPFD (portable jig
only), IR leaf temperature (Pro only), per-rack flow (Pro diagnostic option), per-rack kWh
metering (absent), pH/EC/water temperature (room-shared at TK-01/trough — these belong to the
terrace and water-skid nodes, never to a rack profile), imaging (nothing installed), any
per-tier climate *control*.

### 1.3 Capability-advert consequences for the HLD

1. **Two truths in one profile.** ADR-0002 says the profile advertises what the device *can
   perform*. The hardware genuinely has `valve.control`, `irrigation.flood`, `airflow.speed`,
   `light.dim` per tier. Phase 1 firmware must not drive them. Recommend the profile schema
   carry, per capability, both `hardware: present|absent|optional` and
   `firmware_commands: [...]` (what *this firmware version* will accept). Firmware v0
   advertises the real hardware inventory with `firmware_commands` = `sys.ping`, `sys.identify`
   only; everything else acks `REJECTED(code=NOT_SUPPORTED_IN_FW_VERSION)`. This keeps the
   registry truthful about the rack for R2, and keeps the console free of controls that
   would violate `absent-by-design.md` assertion 1 *and* the Phase 1 "no actuation" lock.
2. **Standard vs Pro profiles are separate profiles**, not a flag: `rack-a.grow.std` (1× T/RH
   at tier 3) and `rack-a.grow.pro` (4× T/RH per tier, optional IR leaf). Phase 1 ships
   only `std` as the virtual profile; `pro` is not to be invented until the hardware program
   publishes it.
3. **GROW-S variant warning.** Rack A Spec §08 p.17 lists a de-scoped "GROW-S" build (1 fill
   solenoid, single header drain valve, **no rack controller**, room T/RH only) and remarks
   "Build this at Ooty. The room controller drives two racks directly — a per-rack controller
   earns its place at eight racks, not two." The contracts (which win, rule 5) say 11 rack
   controllers, so the profile above is correct — but if the hardware program's first Ooty
   racks are actually GROW-S, the first real device the firmware meets will have no ESP32
   at all and the "≥1 real rack controller if hardware ready" acceptance criterion changes
   shape. **Ask the hardware program which build the first physical racks are.**
4. **Tag uniqueness.** `LSH-04` is one tag in the contracts but there is one switch per rack
   (11 physical devices); `LD-01..LD-11` are per-rack. The rack T/RH node has **no tag** in
   `instrument-tags.md`. Key telemetry by `(rack_id, channel)` and treat instrument tags as
   metadata, never as the unique key; feed both gaps back to the contracts review.

---

## Q2. Sensor class, precision, accuracy per sensed channel

The contracts state plainly: "Sensor ranges, accuracy classes and calibration intervals per
tag — Partially in the Phase D BOM; not consolidated" / "not yet published". So per-tag
figures are **[unknown]**; the class-level Phase D figures below are the only sourced
precision data and should populate the registry's *calibration metadata* (device-control-model
§7), not be hard-coded as plausibility limits.

| Channel | Sensor class | Accuracy (class) | Range | Resolution to carry in schema | Sample/response | Unit | Notes |
|---|---|---|---|---|---|---|---|
| Air temperature | "Digital sensor (SHT-series or equivalent)", Sensirion | **±0.3 °C** (Phase D §4 **[sourced]**, class-level) | Per-tag **[unknown]**. device-control-model §6 uses "−40..85 °C class" as the plausibility placeholder **[sourced as placeholder]** | 0.01 °C (2 dp) — the SHT digital word resolves finer than its accuracy; do not truncate to 0.1 in transport, let the console round | 30 s cadence is ample; sensor τ63 is seconds, the shielded air mass is minutes **[general]** | °C — **not defined in `conventions/units.md`** (see Q2.1) | Exact part (SHT3x/SHT4x/grade) **[unknown]**; accuracy band depends on part and on T/RH region **[general]** |
| Relative humidity | same node | **±2 %RH** (Phase D §4 **[sourced]**) | 0–100 %RH physical | 0.01 %RH (2 dp) in transport | as above | %RH — not in `units.md` | SHT-class ±2 % figure is typically the mid-band spec; wider at extremes **[general]** |
| VPD (computed, never sensed) | — | **[derived]** from the two above: at 22 °C / 62 %RH (mid-band of the sourced 20–24 °C, 55–70 % setpoints) the propagated uncertainty is ≈ ±0.06 kPa RSS, ≈ ±0.07 kPa worst-case, dominated by the ±2 %RH term | band 0.6–1.0 kPa `safety-rules.json` climate.vpd_kpa **[sourced]** | 0.01 kPa | — | kPa (`units.md`) | Leaf assumed = air in the standard build (Rack A Spec §04 p.9). **Feasibility flag for R7:** any future VPD control demand tighter than ~±0.1 kPa cannot be honoured by this chain; the schema must record formula version (FR-TEL-03) *and* the sensor accuracy class so that is visible later |
| Leak puck | "Floor spot sensor", dry contact | binary | — | boolean + raw contact state | On-change immediate + present in every periodic frame; debounce time **[unknown]**, LLD choice, not a safety threshold (the interlock is hard-wired) | — | Sensing principle (conductive/optical) **[unknown]** |
| Header high-level `LSH-04` | "Float or optical level switch" | binary | — | boolean | as leak | — | |
| Fan tacho ×4 | Pulse output from 120×38 mm EC/DC axial fan, 24 V, ≤5 W, ball bearing, IP54 | — | — | Publish **`tacho_hz`** (raw pulse frequency) and only compute `rpm` once pulses-per-revolution is known | 30 s, or on-change with a 60 s heartbeat | Hz (rpm later) — not in `units.md` | Fan model and pulses/rev **[unknown]** (fan spec exists, part not chosen). "Failed fan" rpm threshold for the mandatory alarm **[unknown]** — Phase 1 reports only; sim fault "fan stopped" = 0 Hz is unambiguous |

**Not on a rack (room/terrace/skid nodes; listed so the schema reserves the right classes):**
CO₂ NDIR 400–5,000 ppm ±50 ppm (Phase D §4, SCD41-class) ×2 (control at 1.5 m, safety at
300 mm); pH 0–14 lab-grade; EC "0–5,000+ µS/cm" in Phase D but **mS/cm per contracts
`units.md`** — schema must use mS/cm; water temperature `TE-01/TE-02` (class **[unknown]**);
`LT-02`, `PDI-01`, `UIT-01`, `FS-01`, floats, floor flood, smoke. None of these are Phase 1
firmware deliverables; the virtual room controller may emit them but only from the sourced
classes above.

### 2.1 Gaps in `conventions/units.md` to feed back to the hardware-side schema review

`units.md` has no unit for **temperature**, **relative humidity**, **fan speed**, or
**contact/boolean state**. Telemetry schema v1 will have to define °C, %RH, Hz/rpm, and
bool; the classification already names a hardware-side review of schema v1 as a Phase 1
output — put these four on that review's agenda so they land in contracts v0.3.0 rather
than living only in our schema.

### 2.2 Telemetry sample fields the firmware must carry (from device-control-model §6–7)

`ts_source`, `seq` (per-device monotonic), `quality ∈ {good, suspect, implausible-rejected}`,
`calibration_id`. Add **`clock_status ∈ {unsynced, bus, sntp}`**: a production rack
controller has no IP path and no battery RTC in any source **[unknown → assume none]**, so
its clock comes from the bus master or is wrong after every boot. `seq` is the ordering
key of record; `ts_source` is advisory until `clock_status ≠ unsynced`. The backend should
timestamp on receipt as well (FR-TEL-01 already preserves source clock; make the receive
clock a separate column).

### 2.3 Physical-layer flags for `pcb-layout-engineer` (not Phase 1 firmware, but the firmware must not preclude the fix)

- **I²C over the rack run.** The T/RH node sits at 1239 mm on the front-right upright; the
  enclosure is top-rear above canopy. That is a ~1.5–2.5 m I²C run **[derived from
  geometry, unverified]**. Plain I²C at that length is marginal **[general]**; the spec's
  phrase "T/RH node" may already imply a local MCU/transmitter. Firmware v0 should put the
  T/RH behind a driver interface so I²C-direct, I²C-with-extender, or UART/RS485 transmitter
  are interchangeable without touching the telemetry path.
- **Relay board polarity and boot state.** Whether the 8-ch relay board is active-high or
  active-low, and which ESP32 GPIOs are strapping/boot-glitch pins, is **[unknown]**.
  Firmware v0 must not initialise, map, or even name the relay GPIOs (compile them out), and
  the pcb work must guarantee that reset/boot/brown-out leaves every relay de-energized
  (fills CLOSED, drains OPEN per `safety-rules.json`). This is a HIL check in R2, not a
  Phase 1 item — but the skeleton must not choose pins now.
- **Leak/LSH-04 interlock path.** Rack A Spec §07 p.16 says "Leak sensor closes the rack's
  fill solenoids directly *through the controller*"; contracts `interlocks.md` says
  "hard-wired … software is not in their safety path". Whether the contact is in series with
  the fill-relay coil supply (true hard-wire) or is a GPIO the firmware acts on is
  **[unknown]** from the published contracts. Irrelevant for Phase 1 (no relays driven);
  **blocking for R2** — the contracts win, so the board must make the fill relays drop with
  no firmware involvement, and firmware merely observes. Raise with the hardware program now.

---

## Q3. Telemetry cadence, payload envelope, TLS on ESP32

### 3.1 Cadence

| Channel | Recommended v0 cadence | Basis |
|---|---|---|
| T/RH | every **30 s** | NFR-PER-01 "typical sensor cadence (30–60 s)" **[sourced]**; staleness max-age example for T/RH is 2 min (device-control-model §6) — 30 s gives four samples per staleness window so one lost frame is not a false stale alert **[derived]** |
| Fan tacho ×4 | every 30 s (with the T/RH frame) | same frame; no reason to differ |
| Leak, LSH-04 | on-change immediately (event) **and** state in every 30 s frame | state-not-event: a missed edge must not leave the console wrong |
| Capability advert | on boot, on claim, on request | device-control-model §4 `event` topic |
| Reported state (`sys.identify` active, uptime, fw version, clock_status) | on change + 60 s heartbeat, retained | `state/reported` retained per §4 |

NFR-PER-01's "[A] cadence confirmed per sensor class in LLD" is answered above for the rack
classes. Room/skid classes (CO₂ 5 min staleness example, pH/EC 60-s rolling means per
recovery batch) are room-controller cadences and out of this note's scope.

**Production nuance the HLD must state:** on the real bus the rack controller does not
"publish" at all — the room controller **polls** Modbus registers (Rack A Spec §07 p.15).
Cadence is then a bus-master property. The firmware v0 sample path must therefore be
*transport-agnostic*: a sample record is produced on a timer and (a) pushed over the
bench/CI MQTT shim, or (b) latched into a register image for the bus master to read. Design
the skeleton around (b)-compatible semantics (latest-value + seq + change flags) even
though only (a) runs in Phase 1.

### 3.2 Payload envelope

- A full rack frame (1 T, 1 RH, 4 tacho, 2 booleans, seq, ts, clock_status, quality,
  calibration_id, fw version) is ≈ 300–500 bytes as compact JSON **[derived]**. Per-metric
  topics (`telemetry/<metric>`) make that ~8 messages of ~100 bytes. Either way,
  11 racks at 30 s is on the order of **3 msg/s facility-wide** **[derived]** — trivial
  for Mosquitto, for a Linux room controller, and for a single ESP32 on the bench.
- **Capability advert is the largest message.** Keep it **≤ 4 KB**. Preferred: advert
  carries `profile_id`, `profile_version`, `profile_sha256`, and the per-channel inventory;
  the full profile document lives in the registry and is published once at claim (or fetched
  by the room controller), not re-sent on every boot.
- MQTT QoS 1 for telemetry, QoS ≥ 1 for `cmd`/`ack` (ADR-0004: command topics never QoS 0).
- Bandwidth on the RS485 bus is not a Phase 1 concern (no real bus); note only that at 14
  nodes the documented 30 s cadence is far inside Modbus RTU capacity at any common baud
  **[general]**; baud rate **[unknown]**, not published.

### 3.3 TLS on ESP32 — constraints the skeleton must respect (bench/HIL/retrofit path only)

Because production rack controllers do not terminate TLS (Q1.1), these constraints apply to
the **bench/CI shim and any HIL run where an ESP32 talks to the broker directly**, and to
the wireless-retrofit exception. They must not be allowed to shape the room controller's
credential design (see Q5). All figures **[general]** — Espressif/mbedTLS class knowledge,
verify on the chosen module and IDF version:

- Classic ESP32 has ~520 KB SRAM of which roughly 300 KB is available heap after the Wi-Fi
  stack; one mbedTLS session with default 16 KB in/out record buffers costs ~35–50 KB of
  heap for the handshake. **One** persistent TLS session is comfortable; reconnect storms or
  a second session are not. Keep exactly one broker connection and back off exponentially.
- Enable asymmetric/dynamic TLS buffers (`CONFIG_MBEDTLS_ASYMMETRIC_CONTENT_LEN`,
  `CONFIG_MBEDTLS_DYNAMIC_BUFFER` or equivalents) — the broker never sends a rack controller
  anything near 16 KB.
- If mTLS (security-architecture §2 "preferred"): use **ECC P-256** device certificates, not
  RSA-2048 — RSA client-auth handshakes on ESP32 take seconds and a large stack; ECDSA is
  cheap. If per-device PSK is the "first-phase compromise", the security architecture
  requires it documented in an ADR when firmware work starts — that ADR is a Phase 1 output.
- Credentials in encrypted NVS / secure element (security-architecture §7); never in the
  image. Flash encryption + secure boot are a productisation item, but the partition table
  should reserve for dual-partition OTA now (system-architecture §6) even though OTA is out
  of Phase 1.
- Long-uptime heap fragmentation is the classic failure; the 72-h streaming acceptance test
  should be run on real ESP32 hardware if any is available, with free-heap published as a
  telemetry channel (`sys.heap_free`) so the test proves it rather than assumes it.
- Time: SNTP is available only on the Wi-Fi shim path. Do not make the firmware depend on
  it (Q2.2).

---

## Q4. Safe no-op command for the commissioning round-trip ack

**Safe — use these:**

| Command | Effect | Why it is safe |
|---|---|---|
| `sys.ping` | No I/O at all; ack with `seq`, uptime, fw version, clock_status | Pure message path. Proves broker → (room controller) → device → ack lifecycle with zero physical effect |
| `sys.identify {duration_s ≤ 60}` | Blinks the controller board's status LED (3.3 V GPIO, inside the IP65 enclosure above canopy) and sets `identify: true` in `state/reported` until auto-expiry | Touches nothing at or below bed level, no relay coil, no 24 V load, no fan/LED/valve output. Time-bounded and idempotent (re-issue extends, never stacks) |

Conditions: (1) the identify LED GPIO must be chosen by `pcb-layout-engineer` to be neither
a strapping pin nor shared with any relay/PWM/0–10 V line; (2) whether an operator can *see*
a board LED through the enclosure is **[unknown]** (no panel indicator is specified) — the
console must therefore show the `identify` reported state so the round-trip is verifiable
even if the LED is not; a door-mounted indicator/light-pipe is a pcb/hardware-program
option, not something to assume; (3) commands are refused on an unclaimed device
(security-architecture §2).

**Not acceptable as the ack test, ever, in Phase 1:** anything on the 8 relay channels
(fill/drain), any fan PWM change (fans run continuously by design; a stop is a crop risk
and an actuation), any LED dim/photoperiod change (48 V actuation; also LED driver
behaviour is [OQ-14]), anything that writes the stored schedule or last-known-good
setpoints. This holds for real hardware in HIL as well as for virtual devices; the Phase 1
scope lock ("no command to a real valve/LED/light") is the authority.

---

## Q5. Things in Phase 1 that could pre-empt OQ-3 — keep the design platform-agnostic

OQ-3 (room controller compute platform → ADR-0007) is open; options A (Linux SBC), C
(industrial PC), D (MCU bus master + Linux supervisor) remain; B (collapse onto the facility
host) is not recommended and would require amending ADR-0001. The following would quietly
decide it:

1. **Making the rack ESP32 the MQTT client of record.** If firmware v0 is built as
   "ESP32 publishes MQTT to the broker" with no abstraction, the room controller degrades to
   a pass-through and option B becomes the path of least resistance by accident. The bench
   MQTT shim must be labelled as such, live behind a `link` interface, and the HLD must
   restate ADR-0004: *no service ever talks to a rack controller directly; the room
   controller is the only path.*
2. **Putting buffering/backfill/local history on the rack ESP32.** Those are room controller
   responsibilities (OQ-3 brief R7/R11) and are precisely what disqualifies ESP32-class for
   the room role. Rack firmware v0 keeps a latest-value register image and a small ring of
   recent samples at most — not a store.
3. **Choosing the device credential type on ESP32 grounds and applying it fleet-wide.** The
   rack controller (bench path) may need PSK/ECC constraints; the room controller (Linux
   A/C) does not. Make credential type a per-device-class attribute in the registry.
4. **Defining the Modbus register map in firmware.** It is an unpublished hardware
   deliverable (contracts rule 5, "Still missing"). Firmware v0 may define an *internal*
   register image; it must not publish or freeze a map, and the HLD should name the map as
   a contracts v0.3.0 dependency for R2.
5. **Virtual room controller shape.** The sim's room controller must be the MQTT client and
   the virtual racks must sit *behind* it (over an in-process "bus"), so that a real one can
   be Linux/Go (A/C) or split (D) without any backend change. If the sim lets virtual racks
   talk to the broker directly for convenience, that convenience will leak into the design.
6. **Time source.** Do not assume SNTP-over-Wi-Fi on rack controllers (production racks have
   no IP path). Accept time from the bus master; `clock_status` field per Q2.2.
7. **Repository layout.** `firmware/rack-controller/` gets the skeleton; leave
   `firmware/room-controller/` empty (or a README pointing at OQ-3) rather than seeding it
   with an ESP-IDF or Linux tree. Language for the room controller is downstream of
   ADR-0007 (the OQ-3 brief already notes Go on Linux reinforces ADR-0006).

Nothing in the firmware v0 scope *requires* any OQ-3 outcome; with the seven guards above
Phase 1 stays neutral.

---

## Safety-rules cross-check (nothing exercised, nothing invented)

- `electrical_fail_safes`: Phase 1 firmware touches no mains, no relay, no ELV actuator.
  Any HIL bench for firmware v0 is a 3.3 V/5 V/24 V logic rig with **no solenoids, no fans
  under control, no LED drivers**; a bench that includes the rack enclosure and 230 V is
  hardware-program scope under the Type A RCBO / ELV rules and is not needed for Phase 1.
- `valve_and_actuator_fail_safe_states`: not exercised. Firmware v0 must not initialise
  relay GPIOs (Q2.3). Fan and LED de-energized states are not in `safety-rules.json` and
  are **not invented here** — flag for the hardware program before R2, when the schedule
  executor and power-restore boot sequence (device-control-model §5) need them.
- G1 EC gate correction (contracts CHANGELOG 0.2.0: 1.5×, not ~1.2×): not Phase 1 scope;
  already routed to the maintainer loop by the classification. Noted, not acted on.
- Thresholds a Phase 1 seed alert may cite from a rack's own channels: none are safety
  thresholds. The only rack-channel alerts that are sourced are *existence* alerts (leak
  tripped, LSH-04 tripped, tacho = 0 → "failed fan raises an alarm"); their severity tier
  mapping is an alerts-design question, but a tripped hard-wired interlock is by definition
  SAFETY-class information (the interlock has already acted; the alert is the mirror), and
  should be labelled as a mirror exactly as the classification says for the CO₂ alarm.

## Open items for the HLD to carry (owner: hardware program unless noted)

| # | Item | Blocks |
|---|---|---|
| H1 | Which build are the first physical Ooty racks — GROW (rack controller) or GROW-S (no rack controller)? | Phase 1 "≥1 real rack" acceptance shape |
| H2 | ESP32 variant/module and framework for the rack controller | HIL only; firmware keeps it in board config |
| H3 | T/RH part number and grade; per-tag range/accuracy/calibration interval | Registry calibration metadata seed (can ship with class-level Phase D values, provenance-marked) |
| H4 | Fan model and tacho pulses/rev; "failed fan" threshold | rpm conversion; fan alarm rule (R2) |
| H5 | Leak/LSH-04: contact-in-coil-circuit vs GPIO — confirm true hard-wire | R2, HIL |
| H6 | Relay board polarity, GPIO boot states | R2, HIL |
| H7 | Fan and LED de-energized states (not in `safety-rules.json`) | R2 power-restore boot sequence |
| H8 | Units for °C, %RH, Hz/rpm, bool added to contracts `units.md`; tag for rack T/RH node; per-rack LSH tagging | contracts v0.3.0 / schema v1 hardware review (software owner: HLD) |
| H9 | Device credential type per device class (mTLS-ECC vs PSK) recorded in an ADR | Phase 1 (software owner) |
| H10 | Modbus register map | R2 real-bus firmware; not Phase 1 |
