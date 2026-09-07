# UX information architecture & workflows

**Status:** proposed — this is the **pre-wireframe** design layer (working rules 18–19).
No high-fidelity design is authorized until this IA and the workflows below are reviewed.
Visual/interaction design (later): Figma. Functional requirements: repository only.
**Deliverable mapping:** UX information architecture of the foundation analysis.

## 1. UX principles (ordered)

1. **Safety visibility over aesthetics.** Safety-relevant states (CO2 alarm, E-stop,
   leak interlock tripped, fail-safe engaged) are visually louder than anything else on
   any screen they appear on, and reachable in one tap from anywhere.
2. **Glance → drill → act.** Three depths: facility (10-second answer), zone (rack/tier
   detail), entity (device/batch/recipe). No important action more than 3 taps deep.
3. **Desired vs actual, always together.** Any setpoint display shows what was asked and
   what is (per last telemetry), with staleness indicator — never just one of the two.
4. **Offline-tolerant.** The operator console assumes connectivity can degrade; it shows
   data age explicitly and never renders stale state as current. Operators act on
   hardware (E-stop, isolators) regardless of what the screen says.
5. **Mobile-first for the operator.** P1 works with a phone in the room; desktop is the
   secondary surface (P2/P3 analytics views live there).
6. **CEA and aquarium are different products.** Shared platform, separate navigation,
   separate surfaces. Nothing aquarium-related appears in CEA navigation (and vice
   versa) until the aquarium suite exists as its own product.
7. **B2B portal is a separate surface entirely** — different persona (P5), different
   trust model, different design. It may read the same backend; it shares no chrome with
   the operator console.

## 2. Personas → surface mapping

| Surface | Personas | Trust/device assumptions |
|---|---|---|
| **Operator console** (responsive web app; mobile-first) | P1 operator, P2 grow manager, P3 owner, P4 technician | Authenticated session; role-scoped |
| **B2B portal** (separate web app) | P5 customers | Authenticated customer accounts; tenant-scoped; read-mostly + ordering |
| **Future: aquarium suite** | P7 | Separate product entirely |
| **API** | machines only | Device identity (mTLS/PSK); never a UI concern |

## 3. Operator console — navigation IA

```
Home (Facility Overview)
├── Today: what needs me
│     · active alarms (sorted by severity, ack state)
│     · batches due (harvest window open, phase transitions pending)
│     · zones off-target (desired vs actual beyond threshold, with duration)
│     · device issues (offline, stale, failing commands)
│     · connectivity & last-sync status (edge ↔ cloud)
│
├── Grow (the physical plant)
│     · Facility → Room → Rack → Tier hierarchy, drill by map or list
│     · Rack view: all tiers, active batches, per-tier environment snapshot
│     · Tier/Zone detail: environment (desired vs actual), schedule running,
│       active batch, device states, telemetry charts, recent events, controls
│
├── Batches (production lifecycle)
│     · board/table by phase; batch detail = timeline of everything that happened
│     · actions: plan, advance phase, record event, harvest, abort
│
├── Recipes
│     · template library (crop-filtered), version history per recipe
│     · editor: capability-aware (only shows what the target zone can do),
│       safety-envelope validated, save-as-version workflow
│
├── Production (P2/P3)
│     · calendar (planned/actual harvests), yield reporting, waste, forecast vs actual
│
├── Inventory & Availability (operator view)
│     · lots (physical), reservations, availability timeline (what can be promised)
│
├── Orders (operator view of portal activity) — appears with Release 5
│
├── Devices
│     · fleet health (P4): controllers, sensors, actuators; online/offline/stale
│     · commissioning flow (claim → verify capabilities → assign → test command)
│     · device detail: identity, firmware, capabilities, recent commands/acks, logs
│
├── Alerts
│     · active + history; ack/resolve workflow; per-alert rule reference
│     · safety alarms pinned to top, cannot be dismissed without a reason
│
├── Analytics / R&D (P2/P3)
│     · saved queries (yield by recipe/rack/variety; resources per kg; correlations)
│
├── Maintenance
│     · device maintenance & calibration due; E-stop test log; commissioning checklists
│
└── Settings
      · users & roles, facilities/rooms, backup & export (one-click), integrations,
        safety-envelope read-only view (from safety-rules.json), system info
```

Changes vs. the brief's suggested list: "Crop/Batches" and "Recipes" split into
**Batches** (operational, time-ordered) and **Recipes** (library, versioned) — different
mental modes; "Production" kept for aggregates/forecast-vs-actual; **Maintenance**
elevated to top level (P4 + safety-test logging are recurring operational work, not a
Settings sub-page); "Inventory" merged with availability since operators think in "what
can I promise", not lots.

## 4. Key operator journeys (wireframe-level scope definitions)

**J1 — Morning check (UC-01).** Open console → Home shows: 0 active alarms, 3 batches in
harvest window, tier B2 temperature 0.8°C above target for 2h (advisory), all devices
online. Total time: under 30 seconds. Everything on Home links one tap deep.

**J2 — Alert triage (UC-02).** Alert arrives (push/notification) → open → card: what
(severity), where (zone/device), since when, what the system already did (interlock
actions), suggested check → ack (with optional note) → act on hardware → resolve with
reason. Safety alerts additionally show "hardware has already made this safe" state (e.g.,
"E-stop engaged; fill solenoids closed") — the console informs, hardware protects.

**J3 — Set up a new batch (UC-09).** Batches → New → pick crop/variety → recipe template
pick (version visible) → tray count → assign tier (shows availability + conflicts) →
review expected harvest/yield → confirm. Pinned recipe version + zone assignment
created. ~1 minute for a practiced operator.

**J4 — Adjust a running zone (UC-05/UC-12).** Tier detail → change setpoint or trigger
extra flood → confirmation dialog showing safety envelope + auto-expiry for overrides →
confirm → desired-state updated, command ack visible, deviation recorded on the batch.

**J5 — Harvest (UC-11).** Batch in harvest window → Harvest → enter weights (pre/post
tare), quality grade, waste → lots created, forecast reconciled, order commitments
checked → done. Mobile-first single-screen flow (operator is standing at the rack).

**J6 — Commission a device (UC-04, P4).** Devices → Add → power on device → it appears in
"unclaimed" → claim → shows advertised capability profile → assign to rack/tier → run
guided test (e.g., LED on 10% — operator confirms visually) → activate.

**J7 — One-click backup (UC-18).** Settings → Backup → status of last auto-backup, big
"Export now" button → versioned bundle downloads (config, recipes, batches, inventory,
orders, telemetry summaries, audit) — no manual multi-step export anywhere.

## 5. Alert UX rules (normative for wireframes)

- Severity tiers: **SAFETY** (red; hardware interlock territory; cannot dismiss without
  reason; shows fail-safe state), **OPERATIONAL** (amber; crop/product at risk: off-target
  environment, stale telemetry, pump anomaly), **ADVISORY** (blue/informational: mild
  deviation, maintenance due).
- Every alert carries: rule id, zone/device path, first-occurrence + duration, current
  value vs threshold, and a link to "what this rule protects" (operator education, from
  safety-rules.json citations).
- Alert fatigue is a safety issue: rule thresholds are per-zone configurable; a zone
  whose alerts are habitually dismissed without action is flagged in maintenance review.

## 6. Design system & tooling decisions

| Artifact | Source of truth | Notes |
|---|---|---|
| Functional requirements, IA, flows, wireframes | **Repository** (`docs/ux/`, `docs/requirements/`) | Reviewable, diffable, agent-readable |
| Low-fi wireframes | Repository (mermaid/ASCII + interaction notes in `docs/ux/wireframes/`) | Produced by `ux-product-designer`; reviewed before any hi-fi |
| Visual design, interaction design, components | **Figma** (when authorized) | Links from repo docs to Figma frames; Figma never introduces scope |
| Design tokens | Figma → exported tokens file in repo (single export path, reviewed in PRs) | Avoids visual drift between Figma and implementation |
| Status/severity semantics | `safety-rules.json` + alert rules in repo | Never redefined in Figma |

Hi-fi Figma work is **explicitly deferred** until this IA + journeys J1–J7 are reviewed and
Release 1 console scope is frozen (roadmap gate).