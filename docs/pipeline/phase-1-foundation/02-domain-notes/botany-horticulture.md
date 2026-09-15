# 02 — Domain note: botany & horticulture — phase-1-foundation

**Pipeline:** `/ship` · **Slug:** `phase-1-foundation` · **Date:** 2026-09-10
**Author:** botany-horticulture-specialist (Product & Domain Council)
**Scope:** microgreens only. Saffron is deferred indefinitely (owner, 2026-09-07/08) and the
aquatic line starts after the first microgreen rollout — neither is scoped here, and the
`saffron_corm_protocol` block and the aquatic photoperiod / >80 % RH figures in the source set
must **not** be seeded into any Phase 1 rule or virtual profile.

## 0. Sources read

| Source | Used for |
|---|---|
| `docs/pipeline/phase-1-foundation/01-classification.md` | Scope, safety-relevance framing, contracts consumption |
| `docs/roadmap/release-plan.md` §Phase 1 (locked 2026-09-08) | In-scope FR IDs, 72 h soak criterion, "no actuation" |
| `docs/requirements/requirements.md` FR-TEL-01..05, FR-ALR-01..06, FR-SIM-01..03 | Requirement text |
| `.github/agentic-rules/safety-rules.json` (2026-09-05) | Every threshold cited below |
| `knowledge/cea/Phase_A_Literature_Review_Facility_Design.md` §1.1–1.7, §2.1, §2.3, §2.4, §2.6 | Crop physiology, Ooty climate |
| `knowledge/cea/Phase_D_BOM_Supplier_Catalog.md` sensor rows (lines 66–69) | Instrument ranges / accuracy classes |
| `knowledge/cea/cad/Rack_A_Specification.ocr.md` §04 Sensors (pp. 8–9), §05 Airflow (p. 11) | Sensor placement, "VPD never sensed", fans run through dark |
| `knowledge/cea/cad/CEA_Room_-_Floor_Plan.ocr.md` §06 Climate (pp. 8–9), §07 Sensing | Setpoint table, CO2 safety, sensor allocation by level |
| `docs/iot/device-control-model.md` §6, §7 | Plausibility-gate principle, quality flags, calibration metadata |
| `trophic-contracts` v0.2.0: `conventions/units.md`, `capabilities/absent-by-design.md`, `sensors/instrument-tags.md` | What is (and is not) sensed per tier; units |
| `docs/adr/` 0001–0009 | No prior ADR touches telemetry channel selection, plausibility bounds, or VPD formula; ADR-0002 (capability model) and ADR-0004 (units on the wire) constrain naming only |

No prior domain note exists for this slug; this is the first entry in `02-domain-notes/`.

---

## 1. Telemetry channels that matter for microgreens in Phase 1, and plausibility bounds (FR-TEL-02)

### 1.1 Governing principle (already in the project's own model — endorsed)

`device-control-model.md` §6 states that range plausibility is judged against **the sensor's
documented range, not crop targets**. That is the biologically correct framing: a rack air
temperature of 30 °C is bad news for a microgreen crop but is a perfectly *plausible* reading,
and rejecting it would hide exactly the excursion the operator needs to see. Crop bands
(§3 below) belong in alert rules, never in the ingest reject gate.

Consequence for FR-TEL-02: the reject bound for each channel should come from the registry's
per-sensor `range` field (§7) once the part is known. The table below gives the best sourced
bound available today and marks where the source set has none.

### 1.2 Channel table

Levels follow the hardware set (Rack A Spec §04; Floor Plan §07; `absent-by-design.md`):
**RACK** (one node per rack, tier 3, standard build), **TIER** (Pro/R&D build only),
**ROOM**, **WATER** (terrace `TK-01` + recovery trough / skid), **PORTABLE** (not streamed).

| # | Channel | Level / tag | Phase 1 telemetry? | Plausibility (reject) bound | Source of bound | Botany relevance |
|---|---|---|---|---|---|---|
| 1 | Air temperature | RACK, tier-3 node (front-right upright, vented radiation shield, mid-depth, 1239 mm) | **Yes** — primary | **Unknown from `knowledge/cea/`.** Phase D gives accuracy (±0.3 °C) but no range. `device-control-model.md` §6 offers "−40..85 °C class" as the example bound; acceptable as the Phase 1 default **only if labelled as project-doc default, to be replaced by the registry range of the chosen SHT-series part**. | Phase D line 69 (accuracy only); device-control-model §6 | Drives VPD; 20–24 °C operating band (Floor Plan §06) |
| 2 | Relative humidity | RACK, same node | **Yes** — primary | **0–100 %** — physically bounded, not a sensor spec. Reject <0 or >100. Values 95–100 % are physically real (condensation) and must be `good`, not `suspect`. | Physics; consistent with device-control-model §6 | Drives VPD; 55–70 % band; night condensation is the disease trigger (Rack A §05) |
| 3 | Air T / RH per tier | TIER (Pro/R&D build only, one node per tier) | **Optional** — only on a virtual profile that advertises the Pro capability | Same as 1–2 | Rack A §04 "Pro / R&D build only, same mounting rule" | Tier-to-tier microclimate research; not standard |
| 4 | VPD (air) | RACK — **computed**, never sensed | **Yes** — computed (FR-TEL-03) | Computed value must be ≥ 0 kPa and ≤ es(T). A negative result means RH > 100 % input; propagate the input's quality flag, do not clamp silently. | Rack A §04 p.9; Floor Plan §06; `absent-by-design.md` assertion 3 | 0.6–1.0 kPa band |
| 5 | Leaf temperature (IR) | TIER, Pro only | **No** (optional Pro; not in the standard profile) | Unknown — no part specified | Rack A §04 | Enables "true" leaf VPD; R&D rack only |
| 6 | Fan tacho | TIER (one EC fan per tier, tacho feedback) | **Yes** — device health, not botany, but botany-relevant: fans must run continuously **including the dark period** (Rack A §05) | **Unknown** — no RPM range or alarm threshold in the source set. "Tacho alarm is not optional" (instrument-tags.md; Rack A §05) but the number is a hardware deliverable. | Rack A §05 p.11; instrument-tags.md | Stagnation at night → leaf condensation → disease |
| 7 | CO2 (enrichment) | ROOM — NDIR at 1.5 m, mid-aisle | **Yes** — room-level only. Per-rack/per-tier CO2 is **absent, explicitly rejected** (`absent-by-design.md`) | **400–5,000 ppm** is the *documented range of the specified module* (SCD41 class, Phase D line 66). `device-control-model.md` §6 uses "0–10,000 ppm" as the class example. Use the Phase D instrument range for reject; readings below the module floor are instrument-implausible. | Phase D line 66; device-control-model §6 | 1,000–1,500 µmol/mol enrichment band (Phase A §1.3) |
| 8 | CO2 (safety monitor) | ROOM — second NDIR at 300 mm above floor, independent monitor with fail-closed solenoid | **Yes, mirror only** — see §3.3 | **Unknown** — the safety monitor's own part/range is not stated. Do not assume it is the same module as #7. | Floor Plan §06 p.9 | Human safety (5,000 ppm alarm), not plant |
| 9 | Room air T / RH | ROOM — one at HVAC supply, one at return | **Yes** | As 1–2 | Floor Plan §06/§07; Rack A §04 p.9 | Cross-check pair for the rack nodes (device-control-model §6) |
| 10 | Solution temperature | WATER — `TE-01` in `TK-01` (terrace), `TE-02` in the trough | **Yes** | **Unknown** — no part or range stated. Physics floor/ceiling for liquid water (0–100 °C) is not a sensor range; registry must supply it. | instrument-tags.md; Floor Plan §06 | Operating max 26 °C, hard inhibit 28 °C (dissolved O2 / Pythium) |
| 11 | pH | WATER — `AT-04` (trough); `TK-01` pH (terrace block) | **Yes** | **0–14** — documented probe range (Phase D line 67); also the physical scale for aqueous solutions. | Phase D line 67 | Operating target 5.6–6.2 (Phase A §1.6) |
| 12 | EC | WATER — `AT-03` (trough); `TK-01` EC | **Yes** | **0 – 5 mS/cm ("5,000+ µS/cm")** — documented probe range, open-ended upper bound. Emit in **mS/cm** per `units.md`. | Phase D line 68; units.md | No operating target exists (see §3.4) |
| 13 | Reservoir level `LT-02`, flow totaliser, `FS-01`, `PDI-01`, `UIT-01`, floats, `LSH-04`, leak pucks `LD-01..11` | WATER / RACK | Yes for device health; **no botany content** | Out of my lane — mechanical/electronics specialists | — |
| 14 | PPFD / PAR | PORTABLE — quantum sensor on a jig, 5 points per tier, commissioning + quarterly | **No — not a streamed channel.** A fixed PPFD sensor is "absent — portable only". | `absent-by-design.md` assertion 2; Rack A §04 | 150–210 µmol/m²/s band, CV ≤ 15 %; it is a *commissioning record*, not telemetry |
| 15 | DLI | Computed (FR-TEL-03) | **Conditionally** — see §2.3 | n/a | safety-rules `lighting.dli_mol_m2_day` (TBD) | Derived, never sourced |
| 16 | Canopy air velocity | PORTABLE — hot-wire anemometer, commissioning + quarterly | **No** — not streamed | n/a | Floor Plan §06; Rack A §05 | 0.2–0.5 m/s band; CV thresholds 15 % / 20 % |
| 17 | Per-tray water level, per-rack kWh | — | **Absent by design** — must not appear in any virtual profile or UI | — | `absent-by-design.md` | — |

### 1.3 Rate-of-change and stuck-value gates

`device-control-model.md` §6 asks for rate-of-change and stuck-value gates. The source set
gives **no numeric rate limits** for any channel; those are instrument-physics numbers the
electronics specialist should supply from the chosen parts, not botany. I note only that the
stuck-value noise floor can be taken from the sourced accuracy classes (±0.3 °C, ±2 % RH,
±50 ppm — Phase D lines 66, 69): a sensor reporting bit-identical values well inside its own
noise band for many samples is a device fault, not a stable room.

### 1.4 Is FR-TEL-02 biologically sound as stated?

**Yes**, with one guard: the plausibility bounds must never be tightened to the crop bands
"for convenience". The failure mode to design against is the one §6 already names — a
sensor stuck at a *safe-looking* mid-range value — which crop-band rejection would make
worse, not better. Also endorse: `implausible-rejected` samples are stored with the flag,
never dropped (FR-TEL-02, §6), because a run of implausible values is itself the sensor-fault
signal FR-ALR-05 (R2) will consume.

---

## 2. VPD formula for FR-TEL-03

### 2.1 What the source set says

The source set states **what** VPD is and **how it is obtained** but gives **no formula**:

- Phase A §1.2 defines VPD as "the deficit between the saturated vapor pressure at leaf
  temperature and the actual vapor pressure of the surrounding air".
- Rack A Spec §04 p.9: "There is no such thing as a VPD sensor — VPD is computed from air
  temperature, relative humidity and leaf temperature. In the standard build it is
  calculated from the rack T/RH node **with leaf assumed equal to air**, which is adequate for
  control; true VPD needs the IR leaf sensor and belongs only on the R&D rack."
- Floor Plan §06: "VPD 0.6–1.0 kPa — computed from T and RH — never sensed."
- `absent-by-design.md` assertion 3, and `safety-rules.json` `climate.vpd_kpa.control_note`.

So the sourced facts are: (a) air-VPD with T_leaf = T_air is the standard-build definition;
(b) the unit is kPa (`units.md`); (c) inputs are the rack tier-3 T/RH node.

### 2.2 Formula to implement (standard physics — flagged as *not* from `knowledge/cea/`)

This is a saturation-vapour-pressure identity, not a plant-safety threshold, so stating it
does not violate the no-invention rule — but it **is not in the source set** and the HLD
must cite it as an external standard, not as a project-sourced value. Recommended:

```
es(T)   = 0.6108 * exp( 17.27 * T / (T + 237.3) )      # kPa, T in °C  (Tetens form as
                                                        #  used in FAO Irrigation & Drainage
                                                        #  Paper 56, Eq. 11)
ea      = es(T) * RH / 100                              # actual vapour pressure, kPa
VPD_air = es(T) - ea = es(T) * (1 - RH/100)             # kPa
```

- **Formula version string** (FR-TEL-03 "formula version recorded"): something like
  `vpd_air.tetens_fao56.v1`, with `leaf_temp_assumption = "equal_to_air"` recorded alongside.
  A future Pro/R&D "true VPD" series using the IR leaf sensor (`VPD_leaf = es(T_leaf) − ea`)
  is a **different series with a different version string**, not a silent replacement.
- Tetens, Magnus and Buck coefficients give slightly different es(T); the choice does not
  matter agronomically in 20–24 °C, but it **must be pinned** so stored history stays
  comparable — that is the whole point of recording the formula version.
- Inputs must be the **same sample pair** (same node, same `ts_source`); do not compute VPD
  from a rack T and a room RH. Output quality = worst of the two input qualities.
- If RH > 100 % after the plausibility gate somehow passes, the result is negative: emit with
  `implausible-rejected`, do not clamp to zero.

### 2.3 Unit-test vectors (derived from the formula above; not sourced setpoints)

| T (°C) | RH (%) | es(T) kPa | VPD kPa | Note |
|---|---|---|---|---|
| 20 | 70 | 2.338 | **0.70** | Cool/humid corner of the sourced T/RH bands |
| 20 | 55 | 2.338 | 1.05 | |
| 22 | 65 | 2.644 | **0.93** | A sensible sim steady state — inside all three bands |
| 22 | 62.5 | 2.644 | 0.99 | Band mid-points |
| 24 | 70 | 2.984 | 0.90 | |
| 24 | 55 | 2.984 | **1.34** | Warm/dry corner — **outside** the 0.6–1.0 kPa band |

**Consistency observation (flagged, not a contradiction of `safety-rules.json`):** the
three sourced climate bands (T 20–24 °C, RH 55–70 %, VPD 0.6–1.0 kPa) are not jointly
satisfiable at every corner. At 24 °C the RH must be ≥ ~66 % to stay ≤ 1.0 kPa, and the
0.6 kPa floor is not reachable anywhere inside the T/RH rectangle (minimum is ~0.70 kPa at
20 °C / 70 %). This is normal — VPD is the derived variable and the tighter one — but it
means (i) a Phase 1 alert rule set that seeds all three bands as independent rules will
sometimes raise a VPD advisory while T and RH are both "in band", which is correct
behaviour and should be explained in the rule description, and (ii) the simulator's
steady state must be chosen inside the intersection (e.g. 22 °C / 65 %), not at the
rectangle's corners.

### 2.4 DLI (also FR-TEL-03)

FR-TEL-03 asks for "DLI from light state/photoperiod". `safety-rules.json`
`lighting.dli_mol_m2_day` is **TBD** and explicitly says a derived value must be flagged
as derived. In Phase 1 there is no light actuation and no fixed PPFD sensor, so the only
honest DLI series is:

```
DLI = PPFD_commissioned (µmol/m²/s, from the last portable traverse record for that tier)
      × photoperiod_hours_on (from the declared/simulated light state) × 3600 / 1e6
```

with provenance `derived: commissioning_ppfd × light_state` and the commissioning record's
date attached. If no commissioning PPFD exists for the tier, DLI is **null**, not zero. The
simulator may declare a light state (§4) so the series exercises; it must not invent a PPFD
"reading". The HLD may reasonably defer the DLI series to R2 if the commissioning-record
entity does not exist in Phase 1 — from a botany standpoint that is acceptable, since DLI is
not consumed by any Phase 1 alert.

---

## 3. Alert thresholds — sourced vs TBD (FR-ALR-01 / FR-ALR-06)

Phase 1 alerts are **observational**: they compare telemetry against a band and raise
SAFETY / OPERATIONAL / ADVISORY. Nothing here is a setpoint or control behaviour (R2+).
Every seeded rule must carry the `safety-rules.json` key path as its citation (FR-ALR-06).

### 3.1 Sourced — safe to copy **verbatim** into seed rules

| Rule (band departure) | Value | `safety-rules.json` key | Suggested tier (council decision, not botany fact) | Notes |
|---|---|---|---|---|
| Rack air temperature out of band | 20–24 °C | `climate.air_temperature_c` | ADVISORY (OPERATIONAL if sustained) | Room HVAC owns T; rack node is the observer |
| Rack RH out of band | 55–70 % | `climate.relative_humidity_pct` | ADVISORY (OPERATIONAL if sustained) | The 5 % `hysteresis_pct` is a *dehumidifier control* parameter; reusing it as an alert deadband is reasonable but label it as borrowed from control |
| Computed VPD out of band | 0.6–1.0 kPa | `climate.vpd_kpa` | ADVISORY | See §2.3 consistency note; rule text should say VPD is derived |
| Room CO2 (enrichment NDIR) out of band | 1,000–1,500 µmol/mol | `climate.co2_enrichment_target_umol_mol` | ADVISORY | Only meaningful when enrichment is running; in Phase 1 (no dosing) the sim decides. A "below 1,000" advisory during the dark period is noise — see §4 |
| Solution temperature high | > 26 °C operating max; ≥ 28 °C hard inhibit | `irrigation_fertigation.solution_temperature_c` | OPERATIONAL at 26; **OPERATIONAL, labelled "interlock mirror"** at 28 | The 28 °C action (MV-01 inhibit, G3 hold) is R2+ actuation. In Phase 1 the alert **observes** only; it must state that it is not the interlock |
| Nutrient pH out of operating target | 5.6–6.2 | `irrigation_fertigation.ph_operating_target` | ADVISORY | Both `AT-04` (trough) and `TK-01` pH. Phase A §1.6 also notes < 5 or > 8 stunts growth — that is context, not a second threshold |
| **CO2 safety alarm — mirror** | 5,000 ppm | `climate.co2_safety_alarm_ppm` | **SAFETY** | See §3.3 — mirror of a hardware monitor, never the safeguard |

Fresh-air ACH (0.3–0.5), canopy velocity (0.2–0.5 m/s), velocity CV (15 % / 20 %), PPFD
band (150–210) and photoperiod (16–24 h) are all sourced but **not telemetry streams** in
Phase 1 (portable or commissioning quantities, or actuation state) — no rule can consume
them yet. Do not seed rules with nothing to evaluate.

### 3.2 Sourced but **do not seed in Phase 1**

| Item | Why not |
|---|---|
| pH recovery-water gate G2, 4.5–7.5 (`ph_recovery_water_accept_gate`) | (a) It is a *reuse gate* (process control, R2+), not a telemetry band; (b) `safety-rules.json` itself says the OCR ("45-75") needs a sanity check against the PDF before it drives a code path |
| UV dose 100 mJ/cm², ≥ 70 % rated intensity (G5); filter ΔP 1.0 bar (G4); 4-min P-02 run limit; 8 % blowdown | All reuse-gate / actuation logic; R2+. Telemetry for `UIT-01`, `PDI-01` may stream, but the *gate* semantics are out of scope |
| Every `valve_and_actuator_fail_safe_states` and `electrical_fail_safes` entry | Not exercised by Phase 1 code (classification §Safety relevance) |
| `saffron_corm_protocol.*`; aquatic photoperiod 12–16 h; aquatic > 80 % RH | Out of program scope |

### 3.3 The CO2 alarm is a hardware monitor; software is a mirror

Sourced (Floor Plan §06 p.9; `safety-rules.json` `climate.co2_safety_alarm_ppm` and
`co2_asphyxiation_context`): the 5,000 ppm alarm is an **independent monitor at 300 mm
above the floor with a fail-closed solenoid**; it protects people, not plants, and it works
with the software down. Phase 1 must therefore:

- Seed one SAFETY rule on the *safety-monitor* channel (#8), threshold 5,000 ppm, with
  rule text stating verbatim that it **mirrors** the hardware alarm and is **not** the
  safeguard (classification §Safety relevance item 2; device-control-model §6 "anything
  acting on a single sensor's word for a safety decision is a design violation").
- Not seed a 5,000 ppm rule on the *enrichment* NDIR (#7) as if it were the safety device;
  a cross-check "enrichment NDIR ≥ 5,000 while safety NDIR silent" is a *device-health*
  divergence alert (§6 cross-checks), which the HLD may choose to add.
- Make the rule un-dismissable without a recorded action (FR-ALR-03) — it is the one
  genuine SAFETY-tier rule Phase 1 has.

### 3.4 TBD — must stay **unimplemented and visibly flagged**

| Item | Status in source set | Phase 1 handling |
|---|---|---|
| **EC operating target** | `ec_operating_target` = TBD; Phase A §1.6 names EC a dominant control variable but gives no range | **No EC band rule.** EC streams and is displayed with its instrument range only. Console must not show an "in range / out of range" badge for EC |
| **EC recovery gate G1** | OCR unreliable; contracts CHANGELOG corrects the multiplier to 1.5× setpoint (classification §contracts) | Not Phase 1 (gate). Log for the maintainer loop to reconcile `safety-rules.json` with contracts (contracts rule 5 wins) |
| **DLI target** | TBD | No DLI rule; series (if built) is derived-only, §2.4 |
| **Fan tacho alarm threshold (RPM / % of commanded)** | "Not optional" but no number | Device-health rule can only be "tacho = 0 while fan commanded/expected on" until hardware supplies a band; flag |
| **Alert duration windows** ("threshold + duration semantics", FR-ALR-01) | No source states how long an excursion must persist before it is an alert | Durations are a product/engineering default, not botany. Make them per-rule configurable with the default recorded as `unsourced-default`; I cannot supply them |
| **Staleness max-age** (T/RH 2 min, CO2 5 min) | device-control-model §6, given as "e.g." | Fine as defaults; label as project-doc illustrative, not domain-sourced. Not a plant threshold |
| **Solution temperature low bound** | No source states a minimum | No low-temperature rule. Note Ooty nights of 5–13 °C (Phase A §2.1) will make the terrace `TK-01` read cold; that is real, not a fault |
| **CO2 low bound / ambient floor** | No source | Do not seed a "CO2 too low" rule. A `suspect` heuristic for readings below outdoor ambient would be a reasonable *device-health* cross-check but the floor value is not in the source set — human input needed |
| **Room vs rack T/RH divergence tolerance** | §6 says "divergence → both suspect" without a number | Electronics specialist / human; not botany |

---

## 4. What the virtual rack / virtual room should emit for a meaningful 72 h soak (FR-SIM-01)

Everything in this section is **simulator stimulus**, not a setpoint and not a threshold.
Steady-state values are anchored inside the sourced bands so the seeded rules stay quiet
when nothing is wrong; amplitudes, drift rates and time constants are **illustrative and
unsourced** (FR-SIM-03 physics fidelity is deliberately R6) and must be recorded in the sim
harness as `sim_param`, never copied into a rule. The soak criterion is "72 h, zero gaps
unaccounted" — three full photoperiod cycles is what makes a flat line detectably wrong.

### 4.1 Virtual profiles to run

- **Standard rack profile** (default for 10 of the 11 virtual racks): one T/RH node at
  tier 3, four fan tachos, one leak puck, one `LSH-04`. No PPFD, no per-tier T/RH, no
  per-tier CO2, no tray level (`absent-by-design.md`).
- **Pro/R&D rack profile** (1 of 11): adds four per-tier T/RH nodes (and may advertise the
  optional IR leaf temperature). This exercises the "optional capability" path in the
  registry/console without pretending every rack has it.
- **Virtual room controller**: enrichment NDIR (1.5 m), safety NDIR (300 mm), T/RH supply,
  T/RH return, `TK-01` (TE-01, EC, pH, LT-02), trough (TE-02, AT-03, AT-04).

### 4.2 Diurnal pattern — sourced qualitative shape

| Phase | Sourced basis | Shape the sim should emit |
|---|---|---|
| **Photoperiod** | 16–24 h for microgreens; Zone A 16–20 h (Phase A §1.1, §2.3; `lighting.photoperiod_microgreens_hours`) | Declare a lights-on / lights-off state per rack (no actuation — it is a declared state the sim publishes). Any value inside 16–20 h is defensible; **18 h on / 6 h off** is a mid-band choice, not a recommendation. Stagger rack schedules by a few minutes so the facility view is not one synchronous step |
| **Lights-on: rack air T** | LED sensible load is the room's dominant heat source (Floor Plan §06: sensible 3.78 kW vs latent 1.23 kW); sensors sit ≥150 mm from fixtures in shields precisely because fixtures heat the air (Rack A §04 mounting rules) | Rack tier-3 T rises after lights-on with a first-order lag toward a plateau in the **upper half of 20–24 °C**; room supply T reads slightly below rack, return slightly above supply (the supply/return pair "gives the room's actual duty") |
| **Lights-off: T and RH** | "the dark period, when sensible load collapses but transpiration does not stop with it" — the dehumidifier is sized for this (Floor Plan §06) | T decays toward the **lower half of 20–24 °C**; RH **rises** toward and occasionally **just above 70 %** (e.g. 71–73 % for tens of minutes) so the RH ADVISORY fires and clears; computed VPD falls toward the 0.6–0.7 kPa floor. This is the most agronomically realistic excursion the soak can show |
| **VPD** | Derived (§2) | Never emitted by the sim; the backend computes it. The sim's T/RH pairs should produce VPD mostly in 0.8–1.0 kPa by day and 0.6–0.8 kPa by night |
| **Fan tachos** | "Fans run continuously, including through the dark period" (Rack A §05) | Constant plateau with small noise, day and night. **A tacho that drops at lights-off is a bug in the sim**, not a feature |
| **CO2, enrichment NDIR** | 1,000–1,500 µmol/mol enrichment band (Phase A §1.3); dosing "interlocked with ventilation" (Phase A §2.4) | Phase 1 has no dosing, so the sim chooses: (a) *enrichment-on model* — holds 1,100–1,400 during lights-on with slow ripple, decays toward ambient at lights-off (no enrichment in the dark, plants respire); or (b) *no-enrichment model* — sits near ambient with a shallow draw-down by day and rise by night. Either way the "below 1,000" CO2 ADVISORY (§3.1) will fire every night under model (a); the HLD should gate that rule on declared light state or accept it as the expected nightly advisory. The **ambient floor value is unsourced** — sim must record whatever it uses as `sim_param` |
| **CO2, safety NDIR** | Independent monitor, 300 mm above floor (Floor Plan §06) | Track the enrichment NDIR with a small positive offset (CO2 is denser than air; the low sensor is placed there for that reason) and its own noise. Never emit ≥ 5,000 ppm except under an explicit fault switch (§4.4) |
| **Solution temperature** | `TK-01` on the terrace follows Ooty ambient (5–25 °C annual, nights 5–13 °C — Phase A §2.1); trough `TE-02` is in-room | `TE-01` cool, slow diurnal swing well below 26 °C; `TE-02` tracks room T with a lag. Optionally one afternoon push of `TE-02` to 26–27 °C so the 26 °C OPERATIONAL fires once in 72 h; **do not** cross 28 °C except under a fault switch |
| **pH** | 5.6–6.2 target (Phase A §1.6) | Slow drift inside the band with a small step at each simulated water event; brief excursion to ~6.3 once per soak is a reasonable ADVISORY exercise. Direction and rate of drift are unsourced sim params |
| **EC** | **No target exists** | Emit a constant-plus-noise inside 0–5 mS/cm and label it `unsourced-sim-placeholder`. It must not be used to seed or "validate" any EC rule, and the console must not badge it |
| **Water events** | Daily 02:00 blowdown, first batch dumped (`blowdown_pct_of_returned_volume`; Floor Plan §03) | Optional realism: a small `LT-02` level step and EC/pH nudge at 02:00. This is telemetry mimicry only — no valve is commanded |
| **Tier gradient (Pro rack only)** | No sourced gradient magnitude; the hardware set explicitly designs per-tier fans to *prevent* tier microclimates (Rack A §05) | Small, plausible offsets between tiers (fractions of a degree, a few % RH), upper tiers slightly warmer. Unsourced — sim param |

### 4.3 Noise and cadence

- Noise amplitude can be taken from the sourced accuracy classes: ±0.3 °C, ±2 % RH
  (Phase D line 69), ±50 ppm CO2 (Phase D line 66). Use ~half of those as 1-sigma so
  stuck-value detection has a real noise floor to work against.
- Publish cadence: the source set states none. The staleness defaults in
  device-control-model §6 (T/RH 2 min, CO2 5 min) imply publishing at a comfortable
  fraction of those; the actual interval is an engineering choice (`sim_param`).
- Cross-check coherence: emit rack T/RH, room supply and room return as *related* series
  (shared slow component, independent noise) so the §6 room-vs-rack cross-check has something
  meaningful to compare, and so the "implausible" fault switch produces a visible divergence.

### 4.4 Fault switches (release-plan deliverable 4 — carried as in scope per classification)

| Switch | What to emit | What botany-side rule/gate it should trip |
|---|---|---|
| `offline` | Stop publishing for one rack | Staleness → OPERATIONAL (FR-TEL-04) |
| `stale` | Keep publishing with a frozen `ts_source` / repeated `seq` | Staleness, stuck-value → `suspect` |
| `implausible` | RH = 112 %, T = 130 °C, CO2 = −20 ppm, pH = 15 | FR-TEL-02 `implausible-rejected`, stored not dropped; VPD for that pair must inherit the flag |
| `co2_safety` (recommended addition) | Safety NDIR → 5,200 ppm for a few minutes | The one SAFETY rule; exercises FR-ALR-03 un-dismissable path. Label the injected value as a test in the alert history |
| `stuck_mid_range` (recommended) | T frozen at 22.00 °C with zero noise while room supply/return continue to move | The hard case §6 names: cross-check + stuck detection, not band rules, must catch it |

---

## 5. Requirement soundness — summary for the council

| Requirement | Verdict | Adjustment / missing information |
|---|---|---|
| FR-TEL-02 plausibility | **Sound** | Bounds come from instrument ranges (registry §7), not crop bands. Sourced ranges exist for pH (0–14), EC (0–5+ mS/cm), enrichment CO2 (400–5,000 ppm); RH is physically 0–100 %. **Unknown:** air-T sensor range, water-temperature sensor range, safety-NDIR range, fan tacho range |
| FR-TEL-03 VPD | **Sound** | Formula is standard physics (Tetens/FAO-56), **not** in `knowledge/cea/`; cite it as external, pin a version string, record `leaf = air` assumption. Note the three climate bands are not jointly satisfiable at every corner (§2.3) |
| FR-TEL-03 DLI | **Sound only as a derived series** | Needs a commissioning-PPFD record to exist; otherwise null. Acceptable to defer to R2 |
| FR-ALR-01/06 seed rules | **Sound for the seven sourced bands in §3.1** | Severity tiers and durations are product decisions; EC, DLI, tacho RPM, CO2 floor, solution-T low bound are **unknown** and must not be seeded |
| CO2 5,000 ppm | **Sound as SAFETY mirror only** | Rule text must say it mirrors an independent hardware monitor with a fail-closed solenoid |
| FR-SIM-01 profiles | **Sound** | Standard profile = one T/RH node at tier 3; no PPFD, no per-tier CO2, no tray level. Run one Pro profile to exercise optional capabilities |

## 6. Unknowns I could not source (for human domain-expert input)

1. Air-temperature sensor documented range (part not chosen; Phase D gives accuracy only).
2. Water-temperature element (`TE-01`, `TE-02`) part and range.
3. CO2 *safety* monitor part and range (not necessarily the SCD41-class enrichment module).
4. Fan tacho RPM band / alarm threshold ("not optional", but no number).
5. EC operating target (already TBD in `safety-rules.json`) — and the G1 gate needs the
   contracts-vs-safety-rules reconciliation before R2.
6. Alert duration windows for every band rule.
7. Room-vs-rack T/RH divergence tolerance for the §6 cross-check.
8. CO2 ambient floor for a "suspect below ambient" device-health heuristic.
9. Solution-temperature low bound (none stated; Ooty nights will make `TK-01` read cold).
10. Rate-of-change limits per channel (instrument physics — electronics specialist).

None of these blocks Phase 1, because Phase 1 seeds only observational rules on the seven
sourced bands and rejects only on instrument ranges. Each becomes blocking the moment a
control loop (R2+) is wrapped around it.
