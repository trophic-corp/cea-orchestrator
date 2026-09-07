# Data architecture — relational core, time-series strategy, aggregation

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** initial data architecture of the foundation analysis. Storage
engine choice is [ADR-0004](../adr/0004-mqtt-timescale-postgres-backbone.md).

## 1. Principles

1. **One database engine for operational data** (PostgreSQL + TimescaleDB): the entity
   core, audit, and telemetry hypertables in one transactional store. At this scale
   (≈44 zones, modest sample rates) a second dedicated TSDB is operational overhead with
   no payoff; the hypertable path is well-trodden and the compose skeleton already
   assumes it.
2. **Facts are append-only** where history matters (telemetry, desired-state, audit,
   batch events, forecasts-revised). Updates are allowed only where the value is
   definitionally current-and-correctable (e.g., a device label), and even those keep
   an audit trail.
3. **Provenance is stored with the value, not inferred later** (rd-data-model §2).
4. **Aggregates are derived, recomputable, and cached** — never the primary record.
5. **Schema evolution is versioned and additive**: migrations forward-only per release,
   export/import bundles carry schema versions (FR-BKP-01).

## 2. Stores and what lives where

| Store | Contents | Notes |
|---|---|---|
| PostgreSQL (relational core) | org/facility/room/rack/tier/zone, devices + capability profiles, crops, recipes + versions, batches, harvests, inventory, orders/reservations, users/roles, alert rules + history, audit log | Normalized; org-scoped; strong integrity constraints per domain-model §3 |
| TimescaleDB hypertables | telemetry series (per zone/device/metric), desired-state history, device health events | Raw chunks → space-partitioned by time; retention policies (NFR-DAT-03) |
| Rollup/continuous aggregates | per-zone minute/hour/day stats (min/max/avg, time-off-target, alarm minutes, energy estimates) | The thing dashboards actually read (NFR-PER-02) |
| File storage (host volume, simple) | Export bundles, backups, optional batch photos | Referenced by DB rows with checksums; object store later only if volume demands |
| Edge buffer (controller-local) | Telemetry ring buffer, unacked command log, last desired-state snapshot | Store-and-forward during backend/cloud outage (NFR-REL-02); bounded disk usage with explicit gap markers on overflow |

## 3. Telemetry pipeline (shape)

```
device ──(MQTT/TLS, per-device topic ACL)──► broker (Mosquitto, local)
   └─ sample { device_id, metric, value, ts_source, seq }
broker ──► ingest service: validate (schema, plausibility flag, sequence)
   ├─ write raw hypertable row (quality flag: good | suspect | rejected)
   ├─ update "last known state" table (for dashboards, with sample age)
   └─ evaluate alert rules (streaming, per-zone state machines)
rollups: continuous aggregates per zone/metric (1m, 1h, 1d)
retention: raw 24 months → drop after verified rollup + export window (tunable)
```

Design decisions embedded here:
- **Broker is local** (on the facility host): ingestion survives internet loss entirely;
  a cloud/remote reader is a *separate, optional client* (matches the compose skeleton's
  intent and Rack A Specification §07 local-first architecture).
- **Sequence numbers per device** make replay/ordering gaps detectable (and make
  edge-buffer catch-up verifiable).
- **Plausibility flagging, not dropping** (device-control-model §6): bad data is stored
  flagged; consumers filter by quality.
- Alert evaluation is a **state machine per rule per zone** (threshold × duration ×
  hysteresis), persisted so evaluation survives restarts without re-raising.

## 4. Relational core — notable modeling choices

- Recipes: `Recipe` (name/crop) + `RecipeVersion` (semver label + **content hash**,
  parameters as JSONB with its own schema version). JSONB because recipe parameter sets
  vary by crop/capability and are validated by a versioned JSON-Schema, not by table
  columns. Immutability enforced by trigger/policy: rows referenced by a batch become
  read-only for everyone, no exceptions.
- Capability profiles: JSONB descriptor + `profile_schema_version`; registry keeps an
  index of (device, capability_class, params) rows extracted at provisioning so fleet
  queries ("which zones lack CO2 sensing?") are plain SQL.
- Batches: core columns for lifecycle/status/pinned recipe; events in append-only child
  table; batch "detail" view is a merge of batch row + events + zone assignments +
  harvests.
- Availability/forecast: `HarvestForecast` rows are append-only revisions; the "current"
  forecast is max(revision); confidence parameters live in a versioned policy table
  (so confidence changes are themselves historical).
- Audit: dedicated table, no FK cascade deletes possible (append-only policy), exported
  in every bundle.
- **org_id on every core table + row-level org scoping in the query layer** (ADR-0005).

## 5. Aggregation strategy (facility-level metrics)

The facility overview (FR-FAC-04) and Production/Analytics views are built on:

- **Last-known-state** table (zone × metric → latest value + age) for "now" views —
  never a scan over hypertables.
- **Continuous aggregates** for ranges (charts, "CO2 stats today", off-target minutes).
- **Rollup of rollups** for long ranges (daily → weekly/monthly).
- Resource metrics: water per flood (measured flow or calculated from configured flood
  volume × event count — provenance-marked), electricity (metered or schedule-derived
  estimate), both normalized per batch via zone-assignment windows → per-kg figures.

Facility counts (rooms, racks, active tiers, active batches) are trivial queries over
the core; no caching layer is warranted at this scale.

## 6. Backup & data lifecycle

- Export bundle (FR-BKP-01): schema-versioned manifest + relational dumps (per-table
  JSONL) + rollup summaries + config; checksums per file and for the manifest; one file
  the operator downloads; import validates schema version, migrates forward when
  possible, and refuses incompatible bundles with a clear reason.
- Raw telemetry backup: the bundle carries summaries + last-N-months raw by default;
  a full raw archive is a separate, larger scheduled job (NFR-DAT-03) — explicit
  tradeoff documented rather than silently truncating history.
- Retention changes are **ops-configurable**; anything about to be rolled away can be
  exported first; deletion events are audit-logged.

## 7. What is deliberately NOT in the data architecture yet

- No separate event-stream platform (Kafka etc.) — MQTT broker + outbox pattern to
  PostgreSQL covers this scale; revisit at 4+ facilities or heavy portal traffic.
- No data warehouse / lake — analytics run on the same DB via rollups; revisit with R&D
  release volume.
- No multi-region replication — single facility host + offsite backup copies.
- No per-tenant databases — org-scoping in one schema (ADR-0005).