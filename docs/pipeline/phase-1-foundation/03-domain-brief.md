# 03 — Domain brief: phase-1-foundation (reconciled)

**Reviewer:** systems-integration-reviewer · **Date:** 2026-09-10 · **Pipeline:** `/ship`
**Status: READY** — no unresolved specialist disagreement blocks the HLD. Items marked
**[HLD confirm]** are reconciliations the HLD must adopt or explicitly overturn with a
reason; items in §9 need a human but none blocks Phase 1.

**Inputs reconciled:** `01-classification.md`; `02-domain-notes/` product-strategy,
botany-horticulture, electronics-hardware, iot-control-systems, ux-product-design.
**Spot-checked directly:** `docs/adr/0001, 0002, 0004, 0005, 0008, 0009`;
`docs/iot/device-control-model.md` §3/§4/§6; `docs/roadmap/release-plan.md` Phase 1 (LOCKED
2026-09-08, un-paused 2026-09-10); `.github/agentic-rules/safety-rules.json` (2026-09-05);
`docs/security/security-architecture.md` §2; trophic-contracts v0.2.0 (`README`,
`CHANGELOG`, `sensors/instrument-tags.md`, `capabilities/absent-by-design.md`,
`conventions/units.md`, `safety/interlocks.md`).

Labelling convention carried from the notes: **[sourced]** (document named),
**[proposed default, unsourced]** (engineering choice; may not be cited as a threshold),
**[unknown]** (no source; do not invent).

---

## 0. Where the five notes agree (stated once)

| Constraint | Source |
|---|---|
| Phase 1 actuates nothing. The backend has no `cmd` publish path except the identify class; the sim rejects every other command; firmware v0 initialises no actuator output. | release-plan lock; product §3.3, AC-DEV-03e; electronics Q4; IoT §0/§7 |
| The room controller (virtual in Phase 1) is the **only** MQTT client on the field side. Rack controllers are Modbus RTU slaves in the documented build and never carry an MQTT/TLS stack. | ADR-0004 point 0–1; instrument-tags; electronics Q1.1; IoT §0.1 |
| Virtual devices use the same profile schema and topic/payload schema v1 as real hardware; `sim: true` is the only marker, immutable after claim. | device-control-model §10; product AC-SIM-01a; IoT §1.2 |
| Standard rack profile: one T/RH node at tier 3, leak puck, `LSH-04`, 4 fan tachos. Nothing from `absent-by-design.md` (per-tray level, per-tier CO2, fixed PPFD, per-tier climate control) may be advertised or rendered. | contracts `absent-by-design.md`; Rack A §04; all five notes |
| Plausibility bounds come from the sensor's documented range (advert first, class fallback second), never from crop bands; rejected samples are stored with the flag, never dropped. | device-control-model §6; botany §1; IoT §1.3; product AC-TEL-02 |
| VPD is computed by the backend from the same T/RH sample pair, never sensed or advertised; formula version recorded. | contracts assertion 3; botany §2; electronics Q2; IoT §1.3; UX §4 |
| The CO2 5000 ppm alarm is an independent hardware monitor with a fail-closed solenoid; any software alert on it is a **mirror, not the safeguard**, and must say so. | safety-rules `climate.co2_safety_alarm_ppm`; classification; botany §3.3; IoT §7; UX §6.2 |
| Every seeded rule threshold is copied verbatim from `safety-rules.json` with key path + source string; no rule may carry a `TBD` value; operator overrides are labelled "operator-configured, unsourced". | product AC-ALR-06a; botany §3 |
| `sim/` (or wherever the harness lives) imports nothing from `backend/`, `frontend/`, `firmware/`; coupling is via the broker and `docs/iot/schema/v1/` only. | ADR-0008 rules 1–4; classification; IoT §5.1; product AC-SIM-01c |
| Schema v1 is a Phase 1 output that the hardware program reviews (instrument-tags "hardware must review"); the review agenda includes the units gaps (°C, %RH, Hz, bool). | classification; electronics §2.1/H8; IoT §1.8; product AC-DEL-05 |

---

## 1. Rack-controller Phase 1 transport — RESOLVED

**Positions.** Electronics: the rack controller is a Modbus RTU slave; any MQTT path is a
labelled bench shim behind a `link` interface; design the sample path around
register-image semantics (latest value + `seq` + change flags) even though "only the MQTT
shim runs in Phase 1". IoT: no MQTT/TLS/NTP in the ESP32 image at all; the skeleton emits
v1 payload objects as newline-delimited JSON over UART (`rack-link/0`) to the (virtual)
room controller, which bridges to MQTT and stamps `ts_source`/`clock`/`seq`/`boot_id`.
Product: the HLD must decide; "publish MQTT for dev" is not acceptable if it persists.

**Reconciliation.** These are not in conflict: IoT's `rack-link/0` is a *stricter*
instance of what electronics required. Every guard electronics named (Q5.1 link
abstraction, Q5.2 no buffering on the ESP32, Q5.5 virtual racks sit behind the virtual
room controller, Q5.6 no SNTP dependency, Q2.2 `seq` as ordering key with clock advisory)
is satisfied by `rack-link/0`, and electronics' option (a) "bench MQTT shim" becomes
unnecessary for Phase 1 acceptance.

**Recommendation the HLD adopts [HLD confirm]:**

1. **Phase 1 transport for the ESP32 skeleton is `rack-link/0`**: JSON-lines frames over
   UART (USB-serial on the bench; the RS485 transceiver when fitted), carrying the v1
   payload objects minus `org/site/room/ts_source/clock`, plus `link_seq` and `t_mono_ms`.
   The virtual room controller's `RackLink` adapter has two implementations — in-process
   channel (virtual racks) and serial (real skeleton) — so "≥1 real rack controller if
   hardware ready" is the same test with one adapter swapped.
2. The skeleton's internal sample path is **register-image shaped** (electronics Q3.1(b):
   latest value per channel + `seq` + change flags, produced on a timer), serialised to
   JSON-lines. This is what makes the R2 swap to the Modbus register map a framing change,
   not a firmware redesign.
3. The ESP32 image contains **no MQTT, TLS, or SNTP**. No rack-controller class device
   holds a broker credential in Phase 1.
4. **Direct-MQTT-from-ESP32 is not built in Phase 1.** If `firmware-engineer` needs it for
   bring-up, IoT §1.7's conditions apply (compile-time bench flag, distinct broker
   principal confined to `…/<zone>/<device>/#`, deleted before R2) and electronics §3.3's
   TLS constraints (single session, dynamic buffers, ECC P-256 if mTLS) govern it.
5. `rack-link/0` is an **explicit placeholder with a sunset**: replaced by the Modbus
   register map (contracts "not published"; H10) in R2. The HLD names the register map as
   a contracts v0.3.0 dependency for R2.

**ADR?** **No new ADR.** This is an application of ADR-0004 point 1 ("no service ever
talks to a rack controller directly; the room controller is the only path"), not a change
to it. The HLD records it in a transport section with the sunset condition, and
**ADR-0007 (OQ-3), when drafted, must reference `rack-link/0`** so the placeholder cannot
silently become the production link. If the HLD architect wants the sunset to be
binding rather than descriptive, fold it into ADR-0007's consequences rather than minting
a separate ADR.

**Consequence nobody flagged:** because the only Phase 1 MQTT-client device class is the
room controller, the device-credential decision (§6, ADR-0011) is **decoupled from the
ESP32's TLS constraints**. Electronics Q5.3 ("credential type per device class") is the
right shape; the ESP32 ECC/PSK argument applies only to the bench flag and the
wireless-retrofit exception.

---

## 2. Topic and schema refinements — doc amendment, no ADR

**IoT proposes** `trophic/<org>/<site>/<room>/<zone>/<device>/<leaf>` with `<zone> ∈
{rack-NN, room}`, `<tier>` removed from the topic and carried as payload `scope`, a
retained `capability` leaf, a retained `state/health` leaf with LWT, and a defined
`event` type list; version in payload `v`, not the topic root.

**Against the sources.** device-control-model §4 (status *proposed*):
`trophic/<org>/<site>/<room|rack>/<tier?>/<device>/{telemetry/<metric>, state/reported,
state/desired, cmd, ack, event}`. ADR-0004 point 3 delegates: "Topic taxonomy is versioned
and part of the contract (device-control-model §4)" and gives an inline grammar
(`<rack>/<tier|room>/<device>`) that already differs from §4's own. The ADR's decision
content — `trophic/<org>/<site>` root (ADR-0005 needs the org/site segments), per-device
leaves, retained `state/desired`, `cmd` never QoS 0, room controller as the only field
client — is untouched by IoT's refinement.

**Disposition: amendment to `docs/iot/device-control-model.md` §4, landed as
`docs/iot/schema/v1/` (JSON Schema + golden messages). Neither a new ADR nor a change to
ADR-0004.** The HLD states that ADR-0004 point 3's inline grammar is illustrative and §4
as amended is normative; `docs-writer` updates §4 at ship time. Two things the HLD must
carry with it:

- **ACL wording fix (product vs IoT, neither flagged).** Product AC-ADR-0005a says "a
  device credential cannot publish to another device's topics". Under ADR-0004 the room
  controller publishes *on behalf of* every node it bridges, so its ACL is necessarily a
  room-prefix (`…/<room>/#`), which the literal AC forbids. Restate the AC as: *a principal
  cannot publish outside its ACL prefix; the room-controller principal's prefix is its
  room; a bench/direct principal's prefix is its own device; rack nodes hold no
  credential.* Same intent, correct topology.
- **Ingest cross-check** (IoT §1.1): payload `device_id` vs topic prefix mismatch → stored
  `suspect` + security event, never dropped, never trusted.

---

## 3. Capability advert semantics — RESOLVED, the one real design split

**Positions.** Product 3.4(f) recommends the virtual profile advertise the true Rack A
capability set (fill/drain `valve.control`, `irrigation.flood`, `light.dim`,
`airflow.speed` + sensing) with "no actuation" enforced by the backend (no `cmd` path) and
the sim (`REJECTED: not implemented in v0`). Electronics 1.3.1 proposes the profile carry
both `hardware: present|absent|optional` and `firmware_commands: [...]`, with v0 accepting
`sys.ping`, `sys.identify` only. UX §3.4/§5 renders the actuator inventory in device
detail, and commissioning step 2 checks advertised counts against instrument-tags (4 fill
NC, 4 drain NO, 4 fans, 4 LED pairs, 1 T/RH) and **cross-checks each `valve.control
{failsafe_state}` against `safety-rules.json`, blocking activation on mismatch** — its
acceptance test needs actuator classes in the advert. IoT §0.4 advertises sensing +
`control.compute` + `sys.identify` only, adding actuator classes in R2, so that the
REJECT path is "a real test rather than a stub".

**Reconciliation.** Three specialists (including the hardware truth-owner) need the
inventory advertised; IoT's own advert schema already separates `capabilities` (what is
there) from `commands` (what this firmware accepts), which is exactly electronics' two-field
model. IoT's REJECT test survives unchanged by targeting a class no rack has
(`co2.enrich`, `dosing.fertigation`) for `capability_not_advertised`, and adds a second,
more valuable case: `light.dim` advertised but not in `commands` →
`action_not_supported` (IoT's enum already has it). ADR-0002 point 1 ("classes it can
perform") reads naturally as the hardware inventory; "a control appears only when the
capability exists" (point 2) is satisfied because FR-CTL-06 is R2 and UX renders no
control surface in Phase 1.

**Adopted [HLD confirm]:**

- Profile carries **`capabilities[]` = hardware inventory** (with `hardware:
  present|optional` per electronics) and **`commands[]` = classes/actions this firmware
  version accepts**. Phase 1 `commands` = `["sys.identify", "sys.ping"]` on every profile.
- The **virtual standard rack** advertises the full Rack A inventory: `sense.temp_rh @
  tier-3`, `sense.leak`, `sense.water_level{LSH-04}`, `airflow.tacho ×4`,
  `light.state_report ×4` (sim-declared), plus `valve.control{failsafe: CLOSED} ×4` (fill),
  `valve.control{failsafe: OPEN} ×4` (drain), `irrigation.flood ×4`, `airflow.speed ×4`,
  `light.dim ×4` — none of them in `commands`. `failsafe_state` values are copied
  verbatim from `safety-rules.json.valve_and_actuator_fail_safe_states`.
- The **real ESP32 skeleton** builds its advert from what it detects (T/RH on I²C, dry
  contacts if wired) plus a board-config descriptor for actuator *counts* (no pin mapping —
  see §5.4); on a bare bench board that descriptor is empty.
- **Standard profile only.** Electronics 1.3.2 ("`pro` is not to be invented until the
  hardware program publishes it") wins over botany §4.1's "run one Pro rack": no Phase 1
  AC needs it, and AC-DEV-02b's unknown-class fixture already exercises the optional-
  capability rendering path. Botany's tier-gradient sim params are unused in Phase 1.
- `airflow.tacho` **is Phase 1 telemetry** (product §4, botany #6, electronics Q2, UX tier
  rows all say so; IoT's "tachos join in R2" was tied to `airflow.speed`, which is a
  different class). Metric `fan_tacho_hz`, unit `Hz` (electronics: publish raw pulse
  frequency until pulses/rev is known, H4). UX's "1180 rpm" is illustrative; the console
  renders Hz in Phase 1.

This is why **ADR-0010** exists (§6): the meaning of "advertise" is inherited by every
release and would otherwise be re-litigated in R2.

---

## 4. Proposed defaults — reconciled table

None of these is sourced; none may be cited as a threshold (IoT §9, botany §3.4, product
concern 6). The only sourced 60 s in the source set is the leak-interlock solenoid closure
proof (a valve QC number) and **must not be borrowed** for any timing below (IoT §0.3).

| Parameter | Adopted default | Notes / who disagreed |
|---|---|---|
| Telemetry `interval_s` — T/RH, tacho, switches keepalive, room CO2, water-side | **30 s** | Electronics, IoT agree; product "30–60 s" (NFR-PER-01 [A], LLD confirms per class); botany: engineering choice. Advertised per capability; sim and backend read it from the advert. **[proposed default, unsourced]** |
| Heartbeat (`state/health`) | **30 s** | IoT. Electronics' "60 s reported-state heartbeat" was for the rack controller's own frame, which under `rack-link/0` is not an MQTT heartbeat; rack-node liveness is the bridge's `link_timeout_s`. No conflict once §1 is adopted. |
| `offline_after_s` | **90 s** (3 missed heartbeats) | IoT; product "configured window"; electronics: on the real bus the room controller's Modbus timeout is authoritative, this is the mirror window. |
| `link_timeout_s` (bridge → rack node) | **10 s** | IoT; real value belongs to the Modbus map (R2). |
| Staleness `max_age` | **`max(3 × interval_s, class floor)`**; floors T/RH **120 s**, CO2 **300 s**, switches **120 s** | Product, botany, IoT all adopt device-control-model §6's *example* (T/RH 2 min, CO2 5 min) and all three label it illustrative. Water-side classes (TE, pH, EC, LT-02): **no floor proposed by anyone** — the 3× rule gives 90 s; HLD sets a floor and labels it. |
| Stuck-value N | **20 identical samples → `suspect`** | IoT; botany: noise floor from Phase D accuracy classes (±0.3 °C, ±2 %RH, ±50 ppm); electronics to confirm per class (R2). |
| Rate-of-change gate | **R2** | Botany and IoT both defer; needs per-sensor response figures not in Phase D. |
| `sys.identify.max_duration_s` | **60 s** | Electronics and IoT agree. Enforced at the device. |
| `ack_timeout_s` / `max_retries` / `ttl_s` / `backend_wait_s` / dedup ring | **5 s / 2 / 60 s / 30 s / 32** | IoT only; UX needs only "ack + latency visible". These become the defaults R2 actuator classes inherit unless overridden — record, do not bury. |
| Virtual room controller outbound ring | **6 h at nominal rate (~27 k msgs)**, file-backed | IoT only. Real controller sizes to multi-day (OQ-3 R11). |
| Broker `max_queued_messages` (backend session) | **≥ 2 × ring capacity** | IoT; infra task. Undersizing is caught by gap detection. |
| Air-T plausibility range (virtual profile) | **−40..85 °C**, labelled *project-doc placeholder* | device-control-model §6 example; botany and electronics both say per-part range is **[unknown]**. IoT's example `[-40, 125]` is not adopted (no source). |
| RH plausibility | **0–100 %** (physics) | All agree; 95–100 % is `good`, not `suspect` (botany). |
| Enrichment-NDIR range | **400–5,000 ppm** (Phase D, SCD41 class) | Botany, electronics. Below the module floor = instrument-implausible. |
| **Safety-NDIR range** | **omit from the virtual profile → backend class fallback 0–10,000 ppm, recorded as fallback** | See §5.3 — the bound must exceed 5,000 ppm. Real part range **[unknown]** (botany #3). |
| pH / EC ranges | **0–14** / **0–5 mS/cm open-ended upper** (Phase D) | Emit EC in mS/cm (units.md), never µS/cm. |
| Water-temperature range | **[unknown]** — HLD/LLD must add a class fallback entry, labelled unsourced, and it **must exceed 28 °C** (mirror rule, §5.3) | Botany #2; nobody proposed a number. |
| Sim steady state | **22 °C / 65 % RH** (inside all three climate bands) | Botany §2.3; see §5.5. |

Uncertainty to carry in schema (electronics Q2): propagated VPD uncertainty ≈ ±0.06–0.07 kPa
from the ±2 %RH term; record sensor accuracy class alongside formula version so an R7
control demand tighter than ~±0.1 kPa is visibly infeasible on this chain.

---

## 5. Safety consolidation

### 5.1 Verbatim-copyable seed thresholds (all from `safety-rules.json`, key path cited)

| Rule | Value | Key | Tier (reconciled) | Notes |
|---|---|---|---|---|
| Rack air T out of band | 20–24 °C | `climate.air_temperature_c` | ADVISORY | Observational; HVAC owns T |
| Rack RH out of band | 55–70 % | `climate.relative_humidity_pct` | ADVISORY | `hysteresis_pct: 5` is a dehumidifier control parameter — usable as alert deadband if labelled "borrowed from control". LLD must define hysteresis semantics and check botany's night profile (71–73 % for tens of minutes) actually clears; with a 5 % deadband it may not, which is arguably the truth but must be a known behaviour, not a surprise in the soak. |
| Computed VPD out of band | 0.6–1.0 kPa | `climate.vpd_kpa` | ADVISORY | Rule text states VPD is derived and tighter than T/RH (§5.5) |
| Room CO2 enrichment out of band | 1,000–1,500 µmol/mol | `climate.co2_enrichment_target_umol_mol` | ADVISORY, **seeded `enabled: false`** | Botany: fires every night under any realistic no-dosing model; UX AR-8: noise trains operators to ignore the list. Threshold stays verbatim and visible in Rules; enable text: "when CO2 enrichment is commissioned (R2)". **[HLD confirm]** |
| Solution temperature high | > 26 °C | `irrigation_fertigation.solution_temperature_c.operating_max` | OPERATIONAL | Alarm + G3 hold (not an interlock) |
| Solution temperature hard inhibit | ≥ 28 °C | `…solution_temperature_c.hard_inhibit_c` | **SAFETY (interlock mirror)** | See tier reconciliation below |
| Nutrient pH out of target | 5.6–6.2 | `irrigation_fertigation.ph_operating_target` | ADVISORY | `AT-04` and `TK-01` pH |
| CO2 safety alarm | 5,000 ppm | `climate.co2_safety_alarm_ppm` | **SAFETY (interlock mirror)** | §5.3 |
| Leak puck tripped / `LSH-04` tripped / E-stop or floor-flood (if advertised) | boolean state | contracts `safety/interlocks.md` (hard-wired table) | **SAFETY (interlock mirror)** | Existence alerts, not thresholds |
| Fan stopped | `fan_tacho_hz == 0` | instrument-tags "the tacho alarm is not optional"; Rack A §05 "fans run continuously, including the dark period" | OPERATIONAL | A *state*, not a band. A "tacho low" band is **[unknown]** (H4) — UX's "Fan tacho low — ADVISORY" row cannot exist in Phase 1. |
| Staleness / telemetry gap / device offline / broker disconnected / backup failed | design parameters | cite "design parameter, HLD §x" — **not** `safety-rules.json` | OPERATIONAL | FR-ALR-06 honesty: no key path exists for these |

**Severity-tier reconciliation.** Botany suggested OPERATIONAL for the 28 °C mirror but
explicitly deferred the tier to the council; electronics states the principle "a tripped
hard-wired interlock is by definition SAFETY-class information — the interlock has already
acted; the alert is the mirror"; UX lists all interlock mirrors as SAFETY. contracts
`interlocks.md` lists solution over-temperature > 28 °C in the hard-wired interlock table.
**Adopted: every hard-wired-interlock mirror is SAFETY tier** (undismissable without a
recorded action; console shows "What the hardware has done"). This is an alerting-design
decision, not a threshold, and it is the conservative reading. **[HLD confirm]**

### 5.2 Stays unknown — no rule, visibly flagged

EC operating target (`TBD`) — EC streams with instrument range only; **no in/out-of-range
badge**; sim EC is labelled `unsourced-sim-placeholder`. G1 EC gate — contracts CHANGELOG
0.2.0 corrects the multiplier to 1.5× (contracts win, rule 5); not Phase 1 scope;
maintainer loop reconciles `safety-rules.json`. DLI target (`TBD`). Fan tacho low band.
Alert duration windows per rule (product/engineering default, labelled
`unsourced-default`, per-rule configurable). Solution-temperature low bound (none; Ooty
nights of 5–13 °C make `TK-01` read cold — real, not a fault). CO2 ambient floor
(no "CO2 too low" rule). Room-vs-rack T/RH divergence tolerance (cross-check exists in §6
without a number — Phase 1 cannot flag on it). Rate-of-change limits. Also **not seeded**:
G2 pH gate 4.5–7.5 (reuse gate, and its own OCR caveat), G4/G5/UV/blowdown/run-limit
(process gates, R2+), every fail-safe/electrical entry (not exercised), `saffron_corm_protocol.*`
and aquatic values (out of program).

### 5.3 CO2 mirror-only rule — and a bound-vs-threshold rule nobody stated

- One SAFETY rule on the **safety-NDIR channel** (profile `role: safety-mirror`), threshold
  5,000 ppm verbatim, rule text stating it mirrors the independent monitor at 300 mm with a
  fail-closed solenoid and is not the safeguard. **Not** on the enrichment NDIR.
- Enrichment-NDIR ≥ 5,000 while safety-NDIR silent is a **device-health divergence**
  (device-control-model §6 cross-check), not a second safety rule; HLD may add it.
- **New integration rule:** *for any channel carrying an interlock-mirror rule, the
  plausibility upper bound must exceed the mirror threshold (rule seeding fails otherwise).*
  Botany's 400–5,000 ppm SCD41 range applied to the safety channel would make the
  recommended 5,200 ppm test injection `implausible-rejected` — excluded from alert
  evaluation — and the SAFETY path would fail silently. Same for the 28 °C water-temperature
  mirror against any water-temperature fallback range. For the real safety monitor this is
  also a hardware question: its part must have a documented range comfortably above 5,000
  ppm (botany #3 → hardware program).
- The rule's alert detail quotes `co2_asphyxiation_context` (UX §3.5 "What this rule
  protects") — verbatim from `safety-rules.json`.

### 5.4 Relay drive polarity / GPIO boot state — RESOLVED by deference

IoT §0.5 asked the skeleton to drive relay GPIOs to the de-energised level as the first
boot action, with polarity verified on the bench, "electronics-hardware-specialist to
confirm". Electronics §2.3 answered: polarity and strapping pins are **[unknown]**;
firmware v0 **must not initialise, map, or name relay GPIOs (compile them out)**; the
board must guarantee de-energised at reset/boot/brown-out; polarity/boot-state is an R2
HIL item (H6). Driving pins of unknown polarity is the one way firmware could *energise* a
relay; leaving them untouched keeps the hardware, not software, in the safety path
(contracts `interlocks.md`, ADR-0001 layer 1). No relay board is connected to any Phase 1
bench (electronics safety cross-check).

**Adopted:** electronics' position for Phase 1. The **relay-polarity bench check becomes
the R2 bring-up acceptance item** that gates the first firmware version to map relay pins,
and IoT's "de-energised as first boot action" is recorded as that version's requirement.
Flagged for `security-safety-reviewer` as a settled, conservative choice.

### 5.5 VPD formula citation and the non-jointly-satisfiable bands

- Formula: `es(T) = 0.6108·exp(17.27·T/(T+237.3))` kPa; `VPD_air = es(T)·(1 − RH/100)`.
  **Cite as external standard — FAO Irrigation & Drainage Paper 56, Eq. 11 (Tetens form)
  — not as a project-sourced value.** Version string `vpd_air.tetens_fao56.v1` with
  `leaf_temp_assumption: equal_to_air` (Rack A §04 standard build). A future IR-leaf series
  is a different series with a different version. Inputs: same node, same `ts_source`;
  output quality = worst input; RH > 100 % → negative VPD emitted `implausible-rejected`,
  never clamped. Unit-test vectors in botany §2.3.
- **Alert-engine behaviour for the band inconsistency.** Botany: T 20–24 °C, RH 55–70 %,
  VPD 0.6–1.0 kPa are not jointly satisfiable at every corner (24 °C needs RH ≥ ~66 %; the
  0.6 floor is unreachable inside the T/RH rectangle, min ≈ 0.70 at 20 °C/70 %). **Adopted:
  seed all three verbatim as independent ADVISORY rules** — tightening any band to fit
  would be inventing a threshold (FR-ALR-06). The VPD rule description carries botany's
  explanation ("derived and tighter than T/RH; may raise while both inputs are in band —
  correct behaviour"); the three are **not** root-cause-collapsed under each other; the
  UX noisy-rule stats (`acks_without_action_30d`) and per-zone override (labelled
  operator-configured) are the operator's tools if it chatters. The sim steady state sits at
  22 °C / 65 %, never at a rectangle corner, so the soak is quiet when nothing is wrong
  (product 6.1 item 4).

---

## 6. HLD decisions and ADRs (consolidated from UX flags, product concerns, IoT placement)

### 6.1 ADRs to mint (numbering from ADR-0010; 0007 is reserved for OQ-3)

| ADR | Decision | Why an ADR |
|---|---|---|
| **ADR-0010 — Capability advert semantics: hardware inventory vs firmware-accepted commands; `sys.*` controller command namespace** | Profile = `capabilities[]` (hardware present/optional, ADR-0002 point 1) + `commands[]` (what this firmware accepts). Registers `sys.identify {max_duration_s}` and `sys.ping` in the device-control-model §3 vocabulary as controller-scope, non-actuating commands; the Phase 1 commissioning guided test is `sys.identify` (resolves UX **F-2** and product concern 4a). Rule: a capability in the advert but not in `commands` acks `REJECTED{action_not_supported}` at the edge. | Refines what "advertise" means in ADR-0002 for firmware, sim, registry, console and edge alike; inherited by every release; otherwise re-litigated in R2. |
| **ADR-0011 — Device credential scheme, per device class** | mTLS (security §2 "preferred") vs per-device PSK ("first-phase compromise, documented in an ADR when firmware work starts"). Scope: MQTT-client device classes — in Phase 1 only the room controller (virtual) and any bench principal; rack-controller class holds no broker credential (§1). Credential type is a per-device-class registry attribute (electronics Q5.3). | Security §2 requires it; §6.6 security pass lists it; the §1 resolution removes the ESP32 constraint that was the main argument for PSK. Recommendation only: mTLS with ECC P-256 for anything that ever runs on an MCU. |

Naming: `sys.identify` (electronics, IoT) over `control.identify` (product, UX) — the
`sys.*` namespace keeps controller housekeeping apart from crop-control classes; both need
the same §3 registration, so the choice is cosmetic and settled here.

### 6.2 HLD decisions that do not need an ADR

| # | Decision | Recommendation | Origin |
|---|---|---|---|
| D-1 | Sim harness placement | Top-level **`sim/`**, own Go module, own CI job; virtual room controller at `sim/cmd/virtual-room`; `sim/profiles/*.json` are data, `esp32-skeleton-v0.json` is a *captured* advert; `firmware/room-controller/` stays empty with a README pointing at OQ-3. `sim/` is bound by ADR-0008 rules 1–4 as a fourth subsystem — `docs-writer` adds it to the CLAUDE.md boundary list; not an ADR-0008 supersession (nothing is split). system-architecture §2 already lists `sim` top-level. | classification; IoT §5; electronics Q5.7 |
| D-2 | Topic/schema v1 (§2) | Adopt IoT §1 grammar and envelope; land as `docs/iot/schema/v1/`; amend device-control-model §4; hardware review is a named Phase 1 output. Add `profile_sha256` to the `capability` message (electronics) so registry diffs are cheap; electronics' ≤ 4 KB advert limit applies to the rack-link frame, not the MQTT message. | IoT §1.8; electronics §3.2 |
| D-3 | **F-1** facility structure authoring | Structure (facility → room → rack → tier → zone) is created from a **versioned seed file applied at deploy / via API** (AC-FAC-01b already says "API or seed"); admin edits **FR-FAC-05 metadata only** (position label, tier height, notes, doc links — AC-FAC-05a) in Settings. Full CRUD UI is not a sixth screen; add it only if the HLD wants it. Name it: the commissioning wizard step 3 depends on structure existing. | UX F-1 |
| D-4 | **F-5** DLI | **Ship minimal**, not deferred: a nullable `commissioning_ppfd` attribute on the tier (`operator_entered`, dated), DLI = PPFD × lights-on hours from `light.state_report` × 3600/1e6, `formula_version` recorded, **provenance `estimated`** (UX rule B; product's word) with the input chain on tap; **null when no PPFD exists, never zero**; no DLI rule or badge (`dli_mol_m2_day` is TBD). FR-TEL-03 is in the lock, so deferral would need an owner waiver in `11-e2e-report.md`; botany and UX say deferral is *acceptable*, product's AC-TEL-03b *requires* it — ship-minimal satisfies all three. Virtual racks only (real skeleton has no light state: LED driver [OQ-14]). | botany §2.4; product 3.4(g); UX F-5 |
| D-5 | **F-6** SAFETY UX untestable with the three locked switches | Add a **fourth, test-only sim switch `override`** (set a metric's value; botany's `co2_safety` → safety-NDIR 5,200 ppm for a few minutes) driven via `simctl`/scenario, **never via the device `cmd` schema** (IoT §6). It exists to test FR-ALR-03 and AC-ALR-03a, which *are* in the lock; it is a test fixture, not a scope addition. It runs in its own CI scenario and e2e, **not in the 72 h soak** (product 6.1 item 4: no SAFETY alert from the soak fixture). Injected values are labelled as test in alert history. | botany §4.4; UX F-6; product 6.3 |
| D-6 | **F-7** restore UI vs CLI | **Restore/import is an operator CLI in v0**; the console shows export-now (J7, required), bundle list, validation report, and drill status. Reason beyond surface size: §6.5 restores into a **clean stack with fresh volumes** — there are no user accounts to log into a console with until the bundle restores them, so a console-driven restore cannot perform the locked drill without a bootstrap-admin path. AC-BKP-02a's "operator chooses conflict policy with warnings" is satisfied by the CLI. | UX F-7; product §6.5 |
| D-7 | **F-3** per-zone threshold editing | API `PATCH /alert-rules/{id}` exists (AC-ALR-01a "overridable per zone"); Rules tab **read-only in v0**; overrides labelled "operator-configured, unsourced"; SAFETY rows never editable anywhere. | UX F-3 |
| D-8 | **F-4** notification channel config | Product AC-ALR-04a already puts per-severity channel config in Settings; add a **Notifications section inside Settings** (not a sixth screen). SAFETY channels are additive only. | UX F-4 |
| D-9 | Device-offline alert | Raise **one OPERATIONAL "device offline" alert per device** (IoT, UX) rather than health-state-only (product 3.4(h)). Reason: it is the connectivity-axis twin of the in-scope FR-TEL-04 staleness alert, not FR-ALR-05's *inferred* failure rules; without it the Alerts badge misses an offline rack. Trace it under FR-DEV-04/FR-TEL-04. Under IoT's health model an OFFLINE device's metrics are not STALE, so UX's root-cause collapse (one row, not four) falls out of the state machine. **[HLD confirm]** | product 3.4(h); IoT §2.2; UX §6.3 |
| D-10 | Virtual facility topology for water-side channels | IoT's Phase 1 profile set has no water-side channels; product §4, botany §4.1 and UX Grow all assume `TE-01/TE-02/AT-03/AT-04/LT-02` stream, and electronics says they belong to the terrace/skid/room I/O nodes, never a rack or the room controller. **Adopt virtual I/O-node profiles (`io-terrace`, `io-skid`, `io-room`) behind the virtual room controller**, advertising only the sourced classes; IoT's `<device>` examples already anticipate `io-terrace`. Registry then sees the documented 14-node topology. | product §4; botany §4.1; electronics §1.2; UX §3.3 |
| D-11 | Fault-switch taxonomy | Adopt IoT §6 (matches product AC-SIM-02a: `stale` = connected, telemetry stops). Botany's "frozen `ts_source` / repeated `seq`" is IoT's `clock-frozen` variant (→ `suspect`, not stale). Add an ingest rule: duplicate `(device_id, boot_id, seq)` is idempotently ignored, not stored twice. Botany's `stuck_mid_range` = IoT `implausible/stuck`. Either note's implausible fixture values are fine; they must lie outside the *advertised* range. | botany §4.4; IoT §6 |
| D-12 | `seq` / `ts_source` meaning | `seq` and `boot_id` are the **publisher's** (room controller) per device stream; `ts_source` is the publisher's wall clock; rack-link nodes add `link_seq` + `t_mono_ms`; ingest adds `ts_ingest`; staleness runs on `ts_ingest`. Product AC-TEL-01a's "`ts_source` (device clock)" must not be implemented literally for rack nodes — they have no clock (electronics Q2.2, IoT §4.1). Gap record = product §6.1 tuple + `boot_id` + `status {open, filled, unfillable}` + reason. Backfilled rows never touch live state or alert evaluation. | product §6.1; electronics Q2.2; IoT §4 |
| D-13 | NFR-REL-02 "architecture shape" | Product 3.4(e) reading stands: gap detection + recording **in**; push-on-reconnect replay from the virtual room controller's ring **in** (IoT §4.2 — needed for "zero gaps unaccounted"); pull backfill from a local TSDB **R2** (OQ-3). | product 3.4(e); IoT §4.2 |
| D-14 | Home/Grow rendering of R2 state | No Desired column, no batches/off-target tiles, no flood reported state or flood-events chart (nothing emits it in Phase 1; UX's own "renders only if advertised" rule handles it). Note for UX/LLD: the tier wireframe's "Flood: reported idle" and "Fan tacho low" rows are R2. | UX §3.3; product 3.4(b) |
| D-15 | F-8 / F-9 / F-10 / F-11 | Quarantined shown as a claim state, revoke can wait (F-8); IA doc annotated at docs-writer time (F-9); reviewer check at LLD for CEA/aquarium wording (F-10); list view only, no map (F-11). | UX §9 |
| D-16 | Optional diagnostics | `sys.heap_free` as an optional advertised metric on the real skeleton so a 72 h HIL run proves heap stability rather than assumes it. | electronics §3.3 |

---

## 7. Scope drifts — settled by the lock (recorded, not re-litigated)

| Item | `requirements.md` | Lock | Settled disposition |
|---|---|---|---|
| Fault switches offline / stale / implausible | FR-SIM-02 (S, R2) | Deliverable 4 names them | **In.** Delivered as "FR-SIM-02 partial (R1)"; plus the test-only `override` switch of D-5, justified by FR-ALR-03. Command-failure, reboot, power-restore, E-stop switches stay R2. Maintainer loop annotates `requirements.md`; this pipeline does not edit it. |
| FR-BKP-04 selective export | S, R1 | Not in the locked list | **Out.** Next release. |
| NFR-UX-02 ≤ 30 min orientation | M, R1 acceptance | Not in the locked list | **Not a done-gate.** qa-e2e-validator may run it informally; a fail is a finding. |
| NFR-DAT-02 "no silent gaps" | M, untagged | Realised by the 72 h soak | Effectively in; traceability noted. |
| NFR-UX-01 IA principles | M, untagged | "every number shows source age" | Principles 1–4 bind the five screens. |

---

## 8. Other cross-specialist findings (none blocking; carry into the HLD)

1. **Two-party commissioning over rack-link.** UX step 0 ("power on, appears within ~60 s")
   and step 1 (fingerprint last-4 typed by the technician) work for a real skeleton only if
   the board's fingerprint (e.g. eFuse MAC) travels in the rack-link advert and the bridge
   forwards it unaltered; the room controller cannot attest a bus node's identity. Fine in
   Phase 1 (virtual); an open security item for the real bus in R2 — flag for
   `security-safety-reviewer`.
2. **Envelope `clock` enum.** Electronics' `clock_status {unsynced, bus, sntp}` and IoT's
   `clock {ntp, rtc, unsynced}` describe different nodes. Adopt IoT's (publisher's clock);
   `bus` is unnecessary because rack nodes never stamp `ts_source`.
3. **Load fixture.** NFR-PER-01 "≥ 50 zones" exceeds 11 × 4 = 44; the harness must
   parametrise beyond 11 racks (product AC-SIM-01b already says so).
4. **Units gaps to contracts v0.3.0** (electronics H8, IoT §1.3): `degC`, `pct`, `Hz`,
   `bool`; a tag for the rack T/RH node; per-rack `LSH-04` tagging. Telemetry keyed by
   `(device_id, channel)`, instrument tags are metadata.
5. **RK-A-ROOM Rev 3 → Rev 4** citation in device-control-model (contracts CHANGELOG) —
   docs-writer at ship time.
6. **IA §6 review gate** (product concern 5): the UX note is the low-fi wireframe pass
   AC-DEL-02 requires before LLD; the IA/journey *review* itself is still the owner's, and
   `ux-product-designer` is not wired into `/ship` (open-questions #6). Process, not
   architecture — but it will block step 6 if not scheduled.

---

## 9. Needs a human — cannot be resolved here

| # | Question | Who answers | Blocks |
|---|---|---|---|
| Q-1 | Are the first physical Ooty racks GROW (rack controller) or GROW-S (no rack controller, room controller drives two racks directly)? | Owner / hardware program | Only the *optional* "≥1 real rack controller if hardware ready" acceptance shape. **Not Phase 1 done.** R2. |
| Q-2 | CO2 *safety* monitor part and documented range (must exceed 5,000 ppm; §5.3) | Hardware program | R2 (real device). Phase 1 uses the recorded fallback. |
| Q-3 | Air-T and water-temperature (`TE-01/TE-02`) parts and ranges; fan model, pulses/rev, "failed/low fan" band | Hardware program (H3, H4) | R2 rules; Phase 1 ships class-level Phase D values provenance-marked and `tacho == 0` only. |
| Q-4 | Leak/`LSH-04` wiring — contact in the fill-relay coil circuit (true hard-wire) vs GPIO the firmware acts on | Hardware program (H5) | **R2, blocking** for the first actuating firmware. Not Phase 1. |
| Q-5 | Relay board polarity and ESP32 strapping/boot-glitch pins; fan and LED de-energised states (absent from `safety-rules.json`) | Hardware program (H6, H7) | R2 bring-up (§5.4). Not Phase 1. |
| Q-6 | Modbus register map per node | Hardware program (H10; contracts v0.3.0) | R2 — replaces `rack-link/0`. |
| Q-7 | EC operating target (`TBD`); G1 gate reconciliation (contracts 1.5× vs OCR ~1.2×) into `safety-rules.json` | Domain expert; maintainer loop | R2+. Phase 1 seeds no EC rule and no badge. |
| Q-8 | Room-vs-rack T/RH divergence tolerance; CO2 ambient floor; solution-temperature low bound; per-channel rate-of-change limits | Domain expert / electronics from chosen parts | R2 cross-check and device-health rules. Phase 1 cannot flag on them. |
| Q-9 | IA §6 review sign-off and wiring `ux-product-designer` into `/ship` | Owner | Step 6 (LLD) scheduling, not the HLD. |
| Q-10 | ADR-0010 and ADR-0011 approval | Owner, via the normal ADR path | HLD gate. |

**No specialist disagreement remains that the evidence could not settle.** Every
reconciliation above that touches safety (§5.1 tiers, §5.3 bound rule, §5.4 relay GPIOs,
§5.5 bands) chose the reading that takes no physical action and copies no invented number;
each is marked **[HLD confirm]** so the HLD gate and `security-safety-reviewer` see them
as decisions, not defaults.

---

## 10. Re-verification log (appended at HLD / LLD gates — do not rewrite above)

### 10.1 HLD gate — 2026-09-10 — `04-hld.md` + ADR-0010 + ADR-0011 — **CONSISTENT** (1 must-fix, 2 clarifications, no domain drift)

**Reviewer:** systems-integration-reviewer. **Checked:** `04-hld.md` (831 lines, all
sections), `docs/adr/0010-…`, `docs/adr/0011-…`, against this brief; spot-checked
`safety-rules.json` (lines 24–62, 116–119), ADR-0008 rules 1–4 (lines 94–99), ADR-0004
point 1 (line 29). No HLD or ADR was modified.

| Check | Result |
|---|---|
| (1) Every **[HLD confirm]** adopted or overturned with reason | **Pass.** All 8 (§1 rack-link/0; §3 advert semantics; §5.1 CO2 enrichment `enabled: false`; §5.1 interlock mirrors SAFETY; §5.3 bound invariant; §5.4 relay GPIOs; §5.5 three bands + 22 °C/65 %; D-9 offline alert) are "Adopted" in HLD §10.1 with reasons; none overturned. HLD adds `duration_s = 0` and "non-rejected samples only" to the SAFETY-mirror rule — an alerting-design refinement in the conservative direction (raises sooner, never later), correctly flagged to `security-safety-reviewer` in §7.4/§14. |
| (2) D-1..D-16 and both ADRs as specified | **Pass with one must-fix (item A below).** D-1 §2; D-2 §3.1; D-3 §5; D-4 §4.1; D-5 §4.7; D-6 §5.1; D-7 §3.4/§7.1; D-8 §6; D-9 §4.2; D-10 §3.3; D-11 §4.1 ⑤/§4.7; D-12 §4.1/§4.5; D-13 §4.5; D-14 §14; D-15 §4.2/§14; D-16 §3.3. ADR-0010 matches §6.1 (two lists, `sys.*` namespace, `sys.identify` naming, `hardware: present\|optional`). ADR-0011 goes past this brief's "recommendation only" to a fleet-wide mTLS ECC P-256 decision with `credential_type` per class and rack/I/O nodes `none` — consistent with §1's decoupling and §6.1; the added Mosquitto per-listener PSK/cert exclusivity argument is new evidence, not a contradiction. |
| (3) §4 defaults carry the same label and value | **Pass.** All 18 rows of §4 appear in HLD §10.2 with identical values under the unsourced heading. New HLD defaults are all in §10.2 with the label: water-temperature class fallback 0–100 °C ("liquid-water physics bound, class fallback — not an instrument range"; exceeds 28 °C as §5.3 requires; part range stays [unknown]); band `duration_s` ≥ 2 × `interval_s` (`unsourced-default`, LLD sets); NOISY thresholds; export raw window 30 d; backup 02:00 / 14 bundles; session 12 h / 7 d; 13-rack load fixture; p95. Scenario timings in §4.6 (20 min / 30 min / 1 h) are fixture parameters from IoT §6, not thresholds — acceptable outside §10.2. No HLD value is cited as a threshold. |
| (4) Every **[unknown]** stays unknown | **Pass.** HLD §10.3 reproduces §5.2/§9 with no number filled; the safety-NDIR profile omits `range` (§3.3); "tacho low" row rejected (§11); io-room E-stop/floor-flood omitted unless hardware confirms (§3.3); divergence tolerance recorded as [unknown] (§6). Three new unknowns added (LSH-04 debounce, ESP32 variant, LED visibility) — correct. |
| (5) No new specialist contradiction | **Pass.** Electronics Modbus-slave: §3.2 preserves rack-link/0 with sunset, §7.2/ADR-0011 give rack-controller class `credential_type: none`. Botany bound rule: §4.3 advert-first/class-fallback, ranges never from crop bands, RH 95–100 `good`, negative VPD `implausible-rejected` never clamped. UX value envelope: §3.4 provenance enum includes `estimated` (DLI, D-4) and `operator_entered`; one Vue component. IoT grammar: §3.1 `<room>/<zone>/<device>/<leaf>`, `<zone> ∈ {rack-NN, room}`, tier as payload `scope`, `v` in payload, retained `capability`/`state/health`, LWT, `clock ∈ {ntp, rtc, unsynced}` (§8.2). Seeded key paths in §6 exist verbatim in `safety-rules.json`; `failsafe_state` leading tokens (`CLOSED`, `OPEN`) match lines 118–119. |
| (6) ADR-0008 boundaries | **Pass.** §2 names `sim/` the fourth subsystem (own `sim/go.mod`, own CI job, no `require`/`replace` with `backend/`); §2.1 makes `docs/iot/schema/v1/` and `docs/api/openapi.v1.yaml` contracts of record with conformance tests in all three code subsystems and a spec-first drift check; `boundary-check` CI job enforces rule 3; the multi-stage Docker build composes the console *artefact* at image time without importing source — an acceptable reading of rule 3. §8.2 keeps `sim/` out of `docker-compose.prod.yml` entirely. |

**Inconsistencies to fix (none is a domain or safety-threshold question; the architect
chooses):**

- **A. MUST-FIX — ADR-0010 Decision 1 / Consequences vs Decision 5 and HLD §3.3.**
  ADR-0010 states that a class in `commands[]` but not in `capabilities[]` "is a profile
  validation error" and names the invariant `commands ⊆ capabilities`. Decision 5 and HLD
  §3.3 then put `sys.identify`, `sys.ping` in `commands[]` on **every** Phase 1 profile
  while no profile lists a `sys.*` class in `capabilities[]` (`control.compute` is the
  nearest thing). As written, every Phase 1 profile fails validation. Related: Decision 4
  says `sys.identify` is "bounded by the advertised `max_duration_s`", but Decision 1 puts
  parameters only on `capabilities[]` instances, and HLD §3.3 writes the parameter inside
  `commands[]` (`sys.identify{max_duration_s: 60}`). Two clean fixes, either acceptable
  from the domain side: (i) register `sys.*` as actions on the `control.compute` capability
  instance (so `commands[] = [{class: control.compute, actions: [sys.identify, sys.ping]}]`
  and `max_duration_s` lives on that instance), or (ii) state the invariant as
  `commands ⊆ capabilities ∪ sys.*` with `sys.*` params carried on `control.compute`.
  Fix in ADR-0010 Decisions 1/4/5 + Consequences, mirror in HLD §3.3 and the
  `docs/iot/schema/v1/` profile schema. Blocks the LLD profile schema, not the HLD gate.
- **B. CLARIFY — HLD §6 vs HLD §4.1 ③ and brief §2 ("never trusted").** §6 says SAFETY
  mirrors evaluate on `good`/`suspect` samples; §4.1 ③ stores a topic-mismatched sample as
  `suspect{topic_mismatch}`, which this brief §2 says is "never dropped, never trusted". As
  written, a misrouted or injected message from a claimed device can raise an undismissable
  SAFETY row. Raising is the conservative direction for a mirror (no physical action), so
  this is not a domain contradiction, but HLD §6 should enumerate which `quality_reason`
  values enter SAFETY evaluation (e.g. `stuck`, `clock_anomaly`, `unit_unexpected` yes;
  `topic_mismatch` no, or yes with `is_untrusted_source` on the alert) so the security
  reviewer assesses the nuisance-raise vector deliberately. `unclaimed` rows are already
  excluded by §4.1 ④ (joined to no zone).
- **C. CARRY-FORWARD (minor, LLD).** (a) Brief D-2's "electronics ≤ 4 KB advert limit
  applies to the rack-link frame, not the MQTT message" is not restated in HLD §3.2/§3.3;
  the LLD `rack-link/0` frame grammar should carry it. (b) HLD §3.3 `io-terrace` lists
  `sense.ph`/`sense.ec` untagged while `io-skid` tags `AT-03`/`AT-04`; brief §5.1 cites
  `TK-01` for pH — LLD assigns tags from instrument-tags or marks them [unknown], never
  invents one. (c) The backend publisher allow-list is `{sys.identify, sys.ping}` (§7.2)
  while the API surface is `sys.identify` only (§3.4) and brief §0 row 1 says "except the
  identify class"; `sys.ping` is non-actuating and ADR-0010-registered, so no
  contradiction — the LLD states `sys.ping`'s Phase 1 caller or drops it from the
  allow-list.

**Verdict:** the HLD respects every constraint in this brief; no settled constraint has
drifted; no [unknown] was filled; no new specialist contradiction. Item A is an internal
inconsistency in ADR-0010 that the ADR approval (Q-10) should resolve before the LLD
schema work starts; B and C are clarifications for `security-safety-reviewer` and
`lld-architect` respectively.
