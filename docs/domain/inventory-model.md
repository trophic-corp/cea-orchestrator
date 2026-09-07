# Inventory & forecast availability model

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** inventory model + forecast availability model of the foundation
analysis. Related: [production-model.md](production-model.md) (where lots come from).

## 1. Design principle

Working rule 16: **never treat forecast crop yield as guaranteed inventory.** A tray of
radish seeded today is not stock — it is a *probability-weighted promise* that decays
into certainty as the batch matures. The model separates what physically exists from what
we believe will exist, and every customer-visible availability number is a
risk-adjusted, commitment-netted derivation of both.

## 2. Stock taxonomy (canonical definitions)

| Term | Definition | Source of truth |
|------|-----------|-----------------|
| **Physical stock** | Product physically on hand: harvested, weighed, quality-graded lots | `InventoryLot` records (weighed, not assumed) |
| **Expired / spoiled stock** | Lots past best-before or marked spoiled; subtracted from any availability | lot expiry + loss events |
| **Available stock** | Physical stock − reserved − expired/spoiled | derived |
| **Reserved stock** | Physical stock allocated to confirmed orders | `Reservation` records against lots |
| **Expected production** | Forecast harvests from live batches: date + expected yield | `HarvestForecast` per batch (below) |
| **Committed production** | Part of expected production already promised to orders | reservations against future batch harvests |
| **Forecast stock** | Risk-adjusted expected production − commitments, for a given date | derived via §3 |
| **Unavailable / risk-adjusted stock** | The portion of expected production withheld by confidence policy | derived |

Only **available stock** and **forecast stock** are ever customer-visible; raw expected
yield is an operator/R&D figure, never a portal figure.

## 3. Expected production & confidence

Each live batch carries a `HarvestForecast`:

- **Expected harvest date** — from the pinned recipe version, adjusted by actual batch
  progress (phase transitions re-anchor the date).
- **Expected yield** — from the recipe version's reference yield × tray count, adjusted
  by a **confidence factor**.

Confidence (the part that must not be hand-waved):

```
confidence(batch, t) = base(phase) × health_multiplier × history_multiplier

base(phase):    PLANNED        0.0   (never sellable — nothing planted yet)
                SEEDED         0.25
                GERMINATION    0.50  (survives the germination gate)
                GROWING        0.75  (rises with % of grow days elapsed)
                HARVEST_WINDOW 0.95
health_multiplier: 1.0 normally; <1.0 when the batch has unresolved issue events
                  (crop stress, device failure affecting the zone, off-spec environment
                  duration above thresholds) — severity-scaled, operator overridable
                  with a recorded reason.
history_multiplier: rolling factor from THIS crop/variety's past actual/expected
                  ratio on THIS facility's racks (starts at 1.0, learns from
                  harvested batches; simple statistics, not ML).
```

- The exact base curve numbers above are **placeholders pending domain review** — the
  mechanism is the requirement; the numbers must be reviewed by `botany-horticulture-specialist`
  + Sholaverde's operator experience before Release 5. No number here is sourced from
  `safety-rules.json` or `knowledge/cea/` (they don't exist there — flagged rather than
  invented as fact).
- **Portal availability rule:** a customer may order against forecast stock only when
  `expected_yield × confidence ≥ requested_qty` after existing commitments, with a
  minimum confidence floor (default 0.5 — tunable per crop). Below the floor, the
  quantity simply does not appear as available.
- **Overselling prevention:** reservations are atomic decrements against the forecast;
  a batch abort cancels its forecast and triggers an **order-impact alert** (which orders
  are now under-supplied) — the system never silently re-promises.
- **Actual reconciliation:** at harvest, actual yield replaces the forecast; committed
  quantities are fulfilled from physical stock first; shortfalls surface as
  order-exception workflow, not hidden.

## 4. Orders & reservations

```
Order (portal) ── OrderLine ──► Reservation
                                   ├── against InventoryLot (physical, ships now/near-term)
                                   └── against BatchHarvestForecast (future, ships on harvest date)
```

- Orders have states: `draft → placed → confirmed → (picked | partially-fulfilled | shorted) → delivered | cancelled`.
- Custom packs are OrderLines composed of multiple crops (a pack is a recipe-of-SKUs,
  versioned like everything else so a historical order always shows what it contained).
- Short-fulfillment policy: if actual < committed, allocation is FIFO by order placed
  time unless the operator explicitly reallocates (audit-logged) — recurring customers
  are protected by policy, decided by the operator, not silently by the system.
- Cancellation windows: orders against forecast stock may be cancelled until
  `harvest_date − X` days (X per-crop, operator-configurable, shown to the customer at
  order time).

## 5. Inventory lots

- Created only by harvest records (or manual stock entry with a reason code).
- Carry: crop/variety, batch provenance link, net weight, harvest timestamp,
  quality grade, storage location, best-before (per-crop default, from recipe/lifecycle
  data — microgreens shelf life is short; per-crop defaults are domain input, see
  open questions), pack sizes, loss events.
- Pack/label generation is Release 5 scope (what the customer receives must trace to a
  lot and therefore a batch).

## 6. What is explicitly NOT in this model yet

- Pricing/discounting (business rule, not modeled; OrderLine carries unit price at order
  time for history).
- Delivery/logistics, cold chain, invoicing/payment — deferred until the portal release;
  flagged in the roadmap as explicit non-goals until then.
- Multi-tenant stock pooling — every lot/forecast/order is org-scoped in the schema, but
  no cross-tenant logic exists (ADR-0005).