# OQ-3 — Room controller compute platform: decision brief

**Status:** brief for decision · **Date:** 2026-09-08
**Decides:** [OQ-3](../../decisions/open-questions-risks-next-actions.md) · closes **AR-2**
**Blocks:** real room-controller firmware (R1/R2); currently pausing the OQ-16 spike
**Deciders:** owner + hardware program + `iot-control-systems-engineer`
**Needs:** a new ADR (**ADR-0007**) once decided — this outlives the release

> **Why this is a brief and not an answer.** I searched the entire engineering set and
> knowledge base. **No source document names a compute platform for the room controller.**
> The single SBC reference anywhere in `knowledge/cea/` is Phase D's *"AI vision/edge
> compute hardware — SBC or edge-AI compute module"*, which is a different role (imaging
> analytics, hardware currently ABSENT). Per CLAUDE.md, an unsourced physical fact is a
> flagged unknown, not a default — so this brief frames the decision rather than making it.

## 1. What the room controller must do

Every row is drawn from the accepted docs, not assumed. This list *is* the requirement
spec for whatever platform gets chosen.

| # | Responsibility | Source |
|---|---|---|
| R1 | **Bus master**, Modbus RTU over RS485 multi-drop, **14 bus nodes** | device-control-model §3, §4; audit §3.2 |
| R2 | **Drain-token arbitration** — fires one tier at a time across all 11 racks | Rack A Spec (pump/header sizing depends on it); ADR-0001 |
| R3 | **8-condition water-reuse gate** | system-architecture L3; WRS Rev 3 |
| R4 | **Dosing** control | system-architecture L3 |
| R5 | **HVAC + CO2 setpoint** control; later (R7) CO2 PID and VPD-derived loops | system-architecture L3; release-plan R7 |
| R6 | **Alarms** | system-architecture L3 |
| R7 | **Local time-series DB** — "the room's own black box" | system-architecture §1, §4 |
| R8 | **Local UI**, serving while the backend is down | system-architecture L3, §6 |
| R9 | **Sole edge MQTT client**, MQTT 3.1.1/5 over **TLS :8883** | ADR-0004 |
| R10 | **Validation authority** — capability + safety envelope + interlock checks on every command before actuation | device-control-model §5; ADR-0001 |
| R11 | **Buffer + sequence-verified backfill** across a backend outage | system-architecture §4; NFR-REL-02 |
| R12 | **UPS-fed**, with an orderly-shutdown story | system-architecture L3 |
| R13 | **Fail-safe boot** to de-energized states before schedules resume | device-control-model §5; safety-rules.json |
| R14 | **Signed, dual-partition, staged OTA with rollback** | system-architecture §6 |
| R15 | Runs the room **with total backend and internet loss** | ADR-0001 (the design law) |

**Timing character — the load-bearing observation.** Hard real-time lives in the *rack*
controllers (ESP32: fill/dwell/drain, leak and header interlocks, fan PWM). The room
controller arbitrates on flood-cycle timescales — seconds to minutes, not milliseconds.
**Nothing in R1–R15 requires a real-time OS.** That is what makes general-purpose Linux a
legitimate candidate rather than a compromise.

## 2. Why ESP32-class is disqualified

R7 (local TSDB), R8 (served UI), R9 (TLS MQTT), R11 (multi-day buffer + backfill) on
~520 KB SRAM, concurrently with bus mastering, is not an engineering judgement call — it
is out of class. This is exactly what OQ-3 recorded. The rack controllers stay ESP32; the
room controller is a different kind of machine.

## 3. Options

### A. Linux SBC on an industrial carrier — *e.g.* Raspberry Pi CM4/CM5, DIN-rail mounted
- **For:** comfortably satisfies R1–R15. RS485 via HAT/USB adapter. Real filesystem →
  real TSDB and buffer. Serves R8 trivially. Phase D records SBCs as *"assembled/available
  via Indian distributors"* — no import blocker for Coimbatore. Runs Go natively → the
  room controller and the backend share a language (OQ-16 criterion 6).
- **Against:** SD-card wear is the classic killer for a 24/7 logger — **requires industrial
  eMMC/SSD, not an SD card**, and that must be in the BOM, not discovered later. Needs a
  real OS-update and OTA story for R14 (dual-partition on Linux is a solved but non-trivial
  problem — e.g. an A/B image scheme). Supply continuity of a specific SBC model over the
  program's life is a procurement risk.

### B. Collapse the role onto the facility host (the compose stack box)
- **For:** simplest possible deployment — one machine, no new hardware, no new OTA story.
  Superficially attractive for Phase 1, where the room controller is virtual anyway.
- **Against — and this is disqualifying as stated:** it puts the room controller and the
  backend in **one failure domain**, which directly contradicts the accepted architecture.
  system-architecture §6 promises *"Backend/host fails → room controller continues all
  control + local logging + local UI."* If they are the same box, that row becomes
  self-contradictory, and ADR-0001's design law — *no software layer may create an
  operating state that requires the backend to prevent an unsafe physical condition* — is
  violated by construction. **Choosing B means amending ADR-0001, knowingly, with the
  hardware program's agreement.** It should not be chosen by accident because it looked
  convenient during Phase 1.

### C. Industrial PC / panel PC (x86, DIN or panel mount)
- **For:** everything in A, plus industrial-grade thermals, power, storage, and longer
  supply guarantees. Panel-PC variants make R8's local UI a physical panel in the room —
  arguably what "local UI" meant in the engineering docs. Best fit for R12's UPS story.
- **Against:** highest unit cost and power draw; heaviest for a single 60 m² R&D room;
  procurement lead time in India needs checking against the hardware timeline (next
  action #7).

### D. Split — MCU bus master + Linux SBC supervisor
- **For:** keeps deterministic bus timing on a microcontroller and puts TSDB/UI/MQTT on
  Linux. Defensible if bus timing later proves tighter than §1 concludes.
- **Against:** two devices, two firmware images, two OTA paths, and a new protocol between
  them — real complexity bought against a timing problem that §1 says does not exist.
  **Do not choose D without first measuring** a bus-timing failure on A or C.

## 4. Recommendation

**A or C**, decided on procurement and enclosure grounds rather than software ones — the
software requirements do not separate them. A is the cheaper, faster start; C is the more
industrially honest fit for a room that must run unattended on a UPS.

**B is not recommended** unless the owner explicitly accepts amending ADR-0001. **D is
premature** absent evidence.

## 5. Interaction with the OQ-16 language decision

The owner has settled OQ-16 on fluency: **Go**. This brief does not disturb that, and the
dependency runs one way only:

| OQ-3 outcome | Effect on the Go decision |
|---|---|
| A or C (Linux) | **Reinforces it** — one language across room controller and backend |
| B (facility host) | Neutral — same host, trivially the same language |
| D (split) | Neutral — Go on the SBC, C/ESP-IDF on the MCU, as the rack controllers already are |

**No OQ-3 outcome favours Node/TS.** The Go decision is therefore robust to this question,
which is why ADR-0006 can be drafted now rather than waiting.

## 6. What is actually needed to close this

1. **Hardware program:** is there an existing platform preference, an enclosure/DIN
   constraint, or a procurement channel that decides between A and C?
2. **Owner:** is the "local UI" a physical panel in the room (→ C), or a browser page on
   the LAN (→ A)?
3. **Confirm** the §1 timing conclusion with `iot-control-systems-engineer` before
   dismissing D.
4. Record the outcome as **ADR-0007** and close AR-2.

**Not needed to close this:** anything from the unreliable-OCR set (OQ-2/OQ-4). This
decision touches no safety threshold.
