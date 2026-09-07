# R&D data model — data collected now for analytics later

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** R&D data model of the foundation analysis. Related:
[production-model.md](../domain/production-model.md) §5 (batch↔environment linkage),
[data-architecture.md](data-architecture.md) (storage strategy).

## 1. Design principle

No ML now. But every ML/analytics question in the brief — *which recipe yielded most, how
did temperature affect growth, water/electricity per kg, which rack performs better,
what conditions correlate with quality, what yield to expect from a configuration* — is
answerable only if specific data exists, cleanly typed, with provenance. This document
defines what must be captured **from day one** so nothing needs backfilling later.
Backfilled data is worse than no data: it silently biases every model trained on it.

## 2. Provenance taxonomy (applies to every stored value)

Every quantitative value the platform stores carries a `provenance` tag:

| Tag | Meaning | Examples |
|-----|---------|----------|
| **measured** | A sensor/actuator/meter reported it | T, RH, CO2 readings; LED reported intensity; kWh meter; flow meter |
| **calculated** | Derived by the platform from measured values (formula versioned + stored) | VPD from T/RH; DLI from PPFD×photoperiod; per-kg resource metrics; forecast confidence factors |
| **estimated** | A modeled projection, may be revised | HarvestForecast yield/date; unmetered energy attribution; Y-day weather-adjusted projections |
| **operator-entered** | A human recorded an observation/fact | Harvest weights (until scales are auto), quality grades, seed lots, batch events, issue severity |
| **experimental** | Data produced under a flagged R&D protocol (A/B trial etc.) | Trial batch arms; intentionally varied setpoints |

Rules:
- Calculated values store their **formula/version** at write time (a VPD computed under
  formula v1 vs v2 is not the same series).
- Estimated values are **mutable in place only via a new revision record** (a forecast
  history is kept; the current value is a pointer).
- A measured value that has passed a plausibility gate is stored raw with its quality
  flag (device-control-model.md §6) — bad data is never silently dropped, it is flagged
  at use time.
- Analytics surfaces MUST render provenance visibly (an operator-entered yield and a
  scale-logged yield must never look identical in a chart).

## 3. Data to capture from day one (per release)

**From Release 1 (telemetry foundation):**
- Zone/environment time-series: temperature, RH, CO2 (where sensed), light state +
  intensity, water temperature (where sensed), flood events, device states.
- **Sensor metadata at ingest:** device id, zone, sensor type/model, calibration status
  (device-control-model.md §7), sample timestamp *at the sensor* (not broker time).
- Desired-state history per zone (what the recipe/override asked for, when).
- Device health events (online/offline, reboots, command failures).

**From Release 3 (batches):**
- Batch provenance: pinned RecipeVersion + content hash, zone assignments with time
  windows, substrate + seed lot (optional fields, encouraged), phase transitions with
  actor+timestamp, all BatchEvents (observations, interventions, issues + severities).
- Harvest outcomes: wet weight, quality grade, waste, per-harvest timestamp; actual vs
  expected delta stored explicitly.

**From Release 4 (resources):**
- Water: per-rack metering where hardware allows (capability-dependent); at minimum,
  flood event counts × configured per-flood volume as a **calculated** estimate, marked
  as such; water recovery loop volumes (from the recovery system's batch/gate records).
- Electricity: per-rack metering where hardware allows; otherwise schedule-derived
  estimates (LED wattage × on-time; pump duty) marked **calculated/estimated**.
- Cost attribution: tariff (operator-entered, versioned) → per-batch, per-kg cost —
  always re-derivable from the raw series.

**Later (flagged, designed-for, not built):**
- Imaging data (Phase C product 2.5 direction): per-zone RGB on schedule; stored as
  objects referenced from batch timeline, never as primary analytics input until a
  review protocol exists.
- Airflow/VPD-derived stress indicators once per-zone sensing granularity supports
  canopy-level values.

## 4. Analytical questions → required data (traceability matrix)

| Question (from brief) | Needs | Available from |
|---|---|---|
| Which recipe version produced highest yield? | RecipeVersion pins + harvest actuals per batch | R3 |
| Which LED setting → best yield? | Desired-state light history + batch yields | R2 (desired-state) + R3 (yields) |
| How did temperature affect growth? | Zone T time-series joined via ZoneAssignment + growth/duration outcomes | R1 + R3 |
| Water per kg? | Flood events/metering + harvest weight | R1/R4 |
| Electricity per kg? | Light/pump/fan state history + metering where available | R1/R4 |
| Which rack performs better? | Same recipe across racks → per-rack yield/environment deltas | R3 |
| Which variety performs better? | Variety-tagged batches | R3 |
| What conditions correlate with quality? | Quality grades (operator-entered) + environment history + issue events | R3 |
| Expected yield for a configuration? | All of the above, accumulated | R6+ (forecast model) |

Note the recurring dependency: **ZoneAssignment + desired-state history + harvest
actuals**. Everything else can be enriched later; these three cannot be reconstructed
after the fact. They are the R&D irreducibles and are fully present by Release 3.

## 5. Experiment support (minimal, no ML)

- A batch flagged `experimental` is excluded from operational aggregates (availability,
  standard yield stats) but included in R&D queries, **tagged**.
- Trial structure (A/B arms, control groups) is expressed as batch grouping metadata —
  no separate experiment engine. If R&D outgrows this (R6), the accumulated data is
  already correctly shaped to migrate.

## 6. What NOT to build now (deliberately deferred)

- Feature stores, ML pipelines, model registries — deferred until R8 *and* the data
  above has demonstrably answered R&D questions with plain queries.
- Automated imaging analytics — hardware doesn't exist yet (Phase C product 2.5,
  TRL 4–5) and no review protocol is defined.
- Cross-facility benchmarking (needs the multi-tenant foundation, later releases).