# CEA domain model — entities and relationships

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** CEA domain model of the foundation analysis. This is the unifying
entity map; behavior-specific detail lives in the sibling docs:
[production-model.md](production-model.md) (batches, lifecycle),
[inventory-model.md](inventory-model.md) (lots, forecasts, orders),
[device-control-model.md](../iot/device-control-model.md) (devices, capabilities,
commands), [rd-data-model.md](../data/rd-data-model.md) (observations & provenance).

## 1. Structural hierarchy

```
Organization (tenant boundary — ADR-0005; initially: Sholaverde's operator org)
 └── Facility (physical site; initially: Ooty)
      └── Room (insulated envelope; initially one — the CEA room)
           └── Rack (physical grow unit — Rack A design, N tiers)
                └── Tier (physical growing level)
                     └── Zone (LOGICAL unit of environmental configuration; default 1:1 with a tier)
                          ├── ZoneAssignment (batch × zone × time interval)
                          ├── DesiredState (per capability, with history)
                          └── Telemetry (per capability/metric, time-series)
```

Rules:

1. **Zone is the pivotal abstraction.** Everything environmental — desired state, actual
   state, telemetry, recipe application, batch assignment — targets a zone. Today a zone
   is one tier; the indirection is deliberate (working rule 9): room-scale zones for
   HVAC/CO2 (FR-FAC-06), multi-tier zones, and a future aquarium "tank zone" are all
   changes to *zone mapping*, not to the core model.
2. **Physical vs logical never merge.** A rack/tier is what you can touch (and what
   hardware mounts to); a zone is what configuration controls. Devices are assigned to
   tiers; capabilities surface to zones.
3. **Org scoping is present from day one** (schema-level `org_id`, single tenant) so
   productization (P6 persona, external farms) never requires a data migration.

## 2. Core entities (summary form)

| Entity | Purpose | Key fields / notes |
|---|---|---|
| `Organization` | Tenant boundary | slug, name; single instance initially |
| `Facility` | Site | name, location, timezone (IST default) |
| `Room` | Insulated envelope | name, floor-plan reference link (docs only) |
| `Rack` | Grow unit | model/type (e.g. "Rack A"), position, status |
| `Tier` | Growing level | index, height class, status |
| `Zone` | Control/configuration target | status (active/idle/offline), notes |
| `Device` | Physical networked thing | identity, type, firmware version, claimed state, zone/tier assignment, capability profile ref |
| `CapabilityProfile` | Versioned descriptor a device advertises | schema-versioned JSON; classes in device-control-model §3 |
| `Crop` / `Variety` | Crop taxonomy | aquatic vs microgreen vs other categories (behavioral flags, not hard subclasses) |
| `Recipe` / `RecipeVersion` | Crop configuration, immutable versions | production-model §4 |
| `ProductionBatch` | One grow cycle | production-model §2–3 |
| `HarvestRecord` | Yield facts | production-model §3 |
| `InventoryLot` | Physical stock | inventory-model §5 |
| `HarvestForecast` | Expected production | inventory-model §3 |
| `Order` / `OrderLine` / `Reservation` | B2B sales | inventory-model §4 |
| `AlertRule` / `Alert` | Monitoring | severity tiers; rule source citations |
| `AuditRecord` | Append-only action log | security-architecture §5 |
| `User` / `Role` | Console access | RBAC per security-architecture §2 |

## 3. Relationship invariants (enforced at schema level)

- A `Zone` belongs to exactly one `Tier` today; the model permits future N-tier zones —
  flagged as a schema allowance, not implemented UI.
- A `Device` assignment is to a `Tier`; its capabilities *surface* in that tier's zone.
  (Room-level devices — HVAC, CO2 — assign to the Room and surface in room-level zones.)
- A `ProductionBatch` must pin exactly one `RecipeVersion` (content-hash-verified) at
  start; the pin never changes; re-assignment mid-batch is a recorded batch event, not
  an edit.
- `ZoneAssignment(batch, zone, interval)` intervals must not overlap for a zone
  (enforced); historical intervals are immutable.
- `DesiredState` and `Telemetry` are append-only; corrections are new records.
- `InventoryLot` created only from `HarvestRecord` (or audited manual entry).
- `Reservation` references either a lot or a batch's `HarvestForecast`, never both, and
  never raw expected yield.
- Every core entity carries `org_id`; every query path is org-scoped (ADR-0005).

## 4. Crop-type flexibility (microgreens vs aquatic vs exotic vs saffron)

The domain model does **not** hard-code crop semantics. Crop categories are behavioral
profiles on `Crop`:

- **microgreens** — tray-based, short lifecycle, harvest-window-centric; recipes carry
  germination/blackout phase parameters.
- **aquatic** — propagation-centric (emersed-form nursery per Phase A §1.9), possibly
  long-lived/mother-plant batches rather than seed-to-harvest; may need phase-set
  extension (production-model §7, open question).
- **exotic / other** — undefined until [OQ-8] resolves; model must not block a third
  category.
- **saffron** — **deferred indefinitely (owner, 2026-09-08; [OQ-1] closed). Not active
  scope; build nothing for it.** Retained here only as the worked example that the model
  is genuinely crop-agnostic: were it ever revived, its two-phase temperature protocol
  (`safety-rules.json` `saffron_corm_protocol`, retained sourced data) would express as a
  recipe with lifecycle phases + a temperature-program sub-recipe — **no new entities and
  no schema change**. That is the property this section exists to demonstrate.

The recipe parameter schema is grouped by capability domain (lighting, climate,
irrigation, lifecycle), not by crop — a new crop category is new recipe parameter sets
and lifecycle phase config, not schema changes.

## 5. Time model

- **Timestamps are UTC at storage, facility-local at display** (NFR-UX-03). IST
  (+05:30, no DST) simplifies; the model still stores UTC because telemetry from
  edge devices with RTC drift must be reconcilable.
- **Source-clock timestamps** preserved on telemetry (device sample time), distinct from
  ingest time; clock-drift monitoring per device is a health metric.
- Schedule semantics (photoperiod start, flood times) are defined in facility-local time
  (an operator thinks "lights on at 06:00", not "01:30Z") with explicit DST-free
  simplification documented.

## 6. Identifiers & naming

- Human-facing IDs: `Rack-A`, `A-T3` (tier), `B-2026-09-10-01` style batch numbers —
  stable, speakable in the room ("check A3" beats "entity 4711").
- Internal IDs: opaque UUIDs; human IDs are a unique-per-scope label field, never the
  primary key (racks get renamed; history must not break).