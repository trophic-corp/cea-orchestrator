# Crop / batch / production model

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** crop/production model of the foundation analysis. Related:
[cea-domain-model.md](cea-domain-model.md) (facility & recipe entities),
[inventory-model.md](inventory-model.md) (what a batch turns into).
**Owner decision 2026-09-07:** this engineering set is **draft-stage (the initial
idea)** — statuses below describe the planned build as drafted and may be revised by the
hardware program; the capability-based model (ADR-0002) is designed to absorb those
revisions without schema or core-code changes.

## 1. Design principle

Production history is an **evidence chain**. The value of the R&D program depends on being
able to answer, years later: *what exactly was grown, under what exact configuration, on
which physical equipment, in what environment, and what came out of it.* Every design
choice below serves that: recipes are versioned and immutable once referenced; batches
pin a recipe version; zone assignment links telemetry to the batch for its exact time
window; deviations and interventions are first-class records, not comments.

## 2. Entity model

```
CropCatalog                logical crop/variety taxonomy
 ├── Crop                  e.g. "Radish", "Basil", "Anubias nana"
 │    └── Variety          e.g. "Radish — Red Rambo"; optional level, defaults to crop
 │
Recipe                     e.g. "Radish microgreen — baseline"
 └── RecipeVersion (immutable once referenced by any batch)
      · version: semver-like MAJOR.MINOR (see §4)
      · parameters grouped by capability domain (lighting / climate / irrigation / lifecycle)
      · provenance: cloned-from version, author, changelog line
      · status: draft | published | retired
      · is_template flag (reusable starting point vs. batch-specific)
      · experimental flag (R&D trial protocol linkage)
      · crop/variety association (may be cross-crop if explicitly marked)
      · expected_yield: reference yield + harvest window (advisory, never inventory)
      · safety envelope: hard bounds derived from safety-rules.json (not editable)
      ;
ProductionBatch            one planted cycle of a crop, the central operational record
 ├── pinned RecipeVersion  (immutable reference; captured at batch start)
 ├── ZoneAssignment[]      (tier[s] + time interval; a batch may move/split — record all)
 ├── BatchPhase[]          lifecycle state history (never overwritten)
 ├── BatchEvent[]          observations, interventions, issues, photos, notes
 ├── SubstrateInput        growing medium + seed lot (when tracked)
 ├── HarvestRecord[]       ≥0; wet weight, quality grade, waste, tray count, time
 ├── InventoryLotLink      harvest → inventory lot (see inventory-model.md)
 └── OutcomeRecord         disposition: sold | consumed R&D | waste | disposed, final notes
```

## 3. Batch lifecycle (state machine)

```
        plan                seed/assign           germination done
 PLANNED ──► SEEDED ──► GERMINATION ──► GROWING ──► HARVEST_WINDOW ──► HARVESTED
    │         (in tier,      (blackout/dome      (lights on per       │  (yield recorded,
    │          recipe          phase, if the       recipe;             │   lots created)
    │          pinned)         recipe has one)     daily care)         │
    │                                                            ABORTED ◄─┘ (reason code)
    └──────────► CANCELLED (never started; reason recorded)
```

Rules:
- Phase transitions are recorded as events with timestamp and actor — never edited
  afterwards; corrections are new events.
- A batch may be **aborted** from any active phase with a mandatory reason code
  (contamination, device failure, crop failure, overcapacity, trial-complete, other+note).
  Aborted batches are kept — they are R&D data, especially the failures.
- HARVEST_WINDOW is derived from the recipe's lifecycle days but is *editable* on the
  batch (a batch is its own reality); the deviation from the recipe is recorded.
- Multiple harvests per batch are allowed (cut-and-come-again crops, thinnings).
- Estimated vs actual: PLANNED carries expected yield/date (from the recipe version,
  adjusted by the operator); HARVESTED carries actuals; the delta is stored explicitly —
  it feeds forecast confidence (inventory-model.md §3).

## 4. Recipe versioning rules

1. A `RecipeVersion` is **immutable** once any batch references it (working rule 15).
   Editing a published version is impossible by design; you publish a new version.
2. Version numbering: `MAJOR` for a change that alters a parameter a running batch
   already depends on (safety-relevant or lifecycle-relevant), `MINOR` for advisory
   metadata/notes. A batch pin stores the version **id and a content hash**, so any
   tampering is detectable.
3. Changes to a recipe create a new version with `cloned_from` lineage — the recipe
   history is a tree, not a pile; R&D comparisons walk the tree.
4. A template is just a `RecipeVersion` with `is_template`; applying it to a tier
   creates a batch-scoped instance. Modifying an instance **never** writes back to the
   template — you save-as a new version if it worked (UC-08).
5. Recipe parameters are grouped by capability domain and may include parameters the
   assigned zone cannot actuate. Application to a zone resolves the recipe against the
   zone's **capability profile** (device-control-model.md): applicable parameters are
   pushed as desired state; inapplicable ones are recorded on the batch as "not applied —
   zone lacks capability", never silently dropped.
6. Hard safety bounds (PPFD band, photoperiod bounds, flood depth/dwell, solution
   temperature, CO2 alarm) come from `safety-rules.json` and are enforced at
   application/command time — a recipe asking for something outside the envelope is
   rejected at *save* time (warning) and at *apply* time (hard block).

## 5. Batch ↔ environment linkage (the R&D join)

For every batch, the platform can reconstruct its environment history:

- `ZoneAssignment(batch, tier, t_start, t_end)` — the relational join key.
- Telemetry is stored per zone with timestamps (data-architecture.md); a batch's
  environment history is the telemetry of its assigned zones over its assignment windows.
- Desired-state history (what the recipe + operator asked for at each moment) is stored
  alongside actual telemetry, so deviations are first-class.
- Interventions (extra flood, override, fan boost) are BatchEvents with timestamps —
  they explain discontinuities in the telemetry.

This linkage is what later answers "how did temperature affect growth?" without having
planned the question in advance. It requires nothing exotic — only that zone assignment
and desired-state history are never overwritten.

## 6. Producer-side production records

Beyond individual batches, the platform aggregates (derived, never stored as primary
facts where recomputable):

- Production calendar: batches by day, planned vs actual harvests.
- Yield per tray / per tier / per rack / per m² / per crop / per recipe version.
- Waste and loss reasons (feed-forward to quality review).
- Resource attribution: water and electricity consumption attributed per rack (metered
  where hardware allows — capability-dependent) and, where metering granularity does not
  allow direct attribution, recorded as an **estimate with method noted** (never silently
  presented as measured). See rd-data-model.md §2 for the measured/estimated taxonomy.

## 7. Open modeling questions (flagged, not decided here)

- Whether aquatic plant batches need a different phase set than microgreens (e.g.,
  propagation from mother plants vs seed; transition to submersed form at the customer).
  The state machine above is generic enough to hold both; aquatic-specific phases are an
  open question for the botany specialist. **Owner sequencing decision 2026-09-07:**
  aquatic production starts after the first microgreen rollout — this question is
  deferred to the aquatic phase, not a Release-3 blocker.
- Whether "exotic plants" (currently undefined in any source document) introduces a
  lifecycle shape the generic state machine cannot hold — blocked on [OQ-8].
- Seed lot and substrate lot traceability granularity (full lot tracking vs. optional
  fields). Modeled as optional now; food-safety requirements may make them mandatory
  later (see open questions).