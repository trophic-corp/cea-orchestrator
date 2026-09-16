# 01 — Classification: phase-1-foundation

**Invoked by:** `/ship` · **Date:** 2026-09-10 · **Branch:** `phase-1-foundation`
**Reviewer:** architecture-reviewer

| Field | Value |
|---|---|
| `kind` (primary) | `backend` |
| `kinds` (all touched) | `backend`, `frontend`, `firmware`, `infra` — **not** `hardware`, **not** `docs-only` |
| `change_class` | `greenfield` |
| Command match | **`/ship` is correct.** No mismatch. |
| Safety-relevant | **yes — indirect** (alert-rule semantics and threshold citation only; no actuation path, no fail-safe state, valve, interlock, dosing, CO2 or electrical threshold is exercised by Phase 1 code). `security-safety-reviewer` is mandatory regardless under `/ship`. |
| Escalation | none |

## ADRs referenced

All are **inputs/constraints** to the HLD, not prior coverage of the capability being
built. None of them describes a shipped implementation of registry, ingest, alerts, backup,
console, firmware skeleton, or sim harness — `backend/`, `frontend/`, `firmware/` contain
only `.gitkeep`, and `docker-compose.prod.yml` is fully commented out ("as each subsystem's
first `/ship` pipeline actually lands code").

| ADR | Status | Binds Phase 1 how |
|---|---|---|
| ADR-0001 edge-first autonomy | accepted | NFR-REL-01/02 shape: edge path exists, no cloud dependency created; room controller is the edge MQTT client |
| ADR-0002 capability-based device model | accepted | FR-DEV-02 capability advert; FR-SIM-01 virtual devices are ordinary devices with `sim: true`, no forked logic |
| ADR-0004 MQTT + Timescale/Postgres backbone | accepted | Transport, topic taxonomy (`docs/iot/device-control-model.md` §4), storage; Mosquitto + Timescale in compose |
| ADR-0005 single-tenant now, tenant-ready | accepted | Org-scoping on every repository method; tenant-scoping test suite is an acceptance criterion |
| ADR-0006 Go backend | accepted | Backend language; CI Go job already present (`build-test-deploy.yml`) |
| ADR-0008 monorepo with split triggers | accepted | Boundary discipline: OpenAPI is the backend/frontend contract; firmware/backend couple only via versioned MQTT schema; no cross-directory imports |
| ADR-0009 Vue 3 console | accepted | Frontend framework; CI Node 20 job already present |
| ADR-0003 recipe immutability | accepted | **Not touched** — recipes are R3 and explicitly out of scope |
| ADR-0007 | *reserved for OQ-3* | Not a dependency: room controller is virtual in Phase 1 (owner decision 2026-09-10) |

New ADRs minted by this pipeline must start at **ADR-0010** (0007 is reserved for OQ-3).

## Rationale

Phase 1 is the first code in every subsystem directory: it stands up the platform spine
(registry, MQTT ingest, telemetry store, alerts v1, backup v1, authz/audit) plus the first
operator console, the first firmware skeleton, the first virtual devices, and the first live
compose/deploy path. No ADR or HLD describes an existing implementation of any of these, so
there is nothing to `/extend` and nothing to `/fix`; the accepted ADRs are architectural
constraints the HLD must honour, not decisions that already cover this capability. That is
the textbook `/ship` case, and the release plan and compose skeleton both say so
explicitly. The primary kind is `backend` because the bulk of the in-scope FR IDs
(FR-FAC, FR-DEV, FR-TEL, FR-ALR, FR-USR, FR-BKP, ADR-0005 tests, NFR-PER) land in the Go
service and its schema; `frontend`, `firmware`, and `infra` are each a real deliverable and
downstream steps must fan out to all four implementation specialists. `hardware` is not
touched: no PCB, pinout, or physical device work is in scope, and OQ-3 does not gate this.

## Safety relevance — detail

Phase 1 cannot actuate anything (locked scope: "no command to a real valve/LED/light —
telemetry and registry only"). Nothing in Phase 1 code reads or enforces a valve fail-safe
state, an interlock timing, a dosing/EC/pH gate, the CO2 5000 ppm alarm action, or an RCBO
requirement. The flag is nonetheless **yes (indirect)** for three reasons the
`security-safety-reviewer` should look at:

1. **FR-ALR-01/02/03** introduce the SAFETY / OPERATIONAL / ADVISORY severity tiers and the
   rule that SAFETY alerts cannot be dismissed without a recorded action. The alert-rule
   schema and lifecycle built here will later carry real safety thresholds; getting the
   semantics wrong now is a design flaw R2+ inherits.
2. **FR-ALR-06** requires every alert to link to its threshold's source citation
   (`safety-rules.json` where applicable). Any Phase 1 seed rule that cites a threshold
   must copy it verbatim from `safety-rules.json` — never invent one. The CO2 alarm in
   particular is an *independent hardware monitor with a fail-closed solenoid*; a software
   alert on it is a mirror, not the safeguard, and must be labelled as such.
3. **FR-TEL-02/03** define plausibility rejection and computed VPD (formula version
   recorded). Plausibility bounds are not safety thresholds, but the sim harness's
   "implausible" fault switch and the ingest path must not silently discard
   (device-control-model §6).

The firmware v0 "command ack" and the commissioning "test command round-trips with ack"
acceptance criterion must be satisfied with a **no-op / identify-class command** against a
virtual device, not a fill/drain/LED command. HLD should state this.

## trophic-contracts v0.2.0 — consumption

Phase 1 **does consume** the sibling repo (`/mnt/d/40_Projects/trophic/trophic-contracts`,
read-only) and should pin `v0.2.0` in the HLD per its rule 3:

| Contract | Consumed by | Why |
|---|---|---|
| `sensors/instrument-tags.md` | firmware v0 capability advert, virtual device profiles, sim harness, backend registry | Tag vocabulary (`TE-02`, `AT-03`, `AT-04`, `LD-01..11`, `LSH-04` …), controller topology (11 x ESP32 rack controllers, room controller as bus master / MQTT client, 14 bus nodes), per-rack actuator counts (4 fill NC, 4 drain NO, 8 relays, 1 fan/tier with tacho, 1 LED dim pair/tier) |
| `capabilities/absent-by-design.md` | virtual device profiles, backend capability model, console (Devices/Grow screens) | Virtual profiles must **not** advertise per-tray level, per-tier CO2, fixed PPFD, or per-tier T/RH control; rack T/RH is one sensor at tier 3 by default; VPD is computed, never sensed; "a control must not appear in a UI for a capability the hardware does not have" |
| `conventions/units.md` | telemetry metric schema, console display | Units for EC, PPFD, airflow, CO2, UV dose; tag patterns |

**Not consumed** in Phase 1 (all actuation/process, R2+): `safety/valve-fail-states.md`,
`safety/interlocks.md`, `hydraulic/*`, `geometry/bed-datum.md`, `process/reuse-decision.md`,
`electrical/elv-boundary.md`.

Two items to record downstream, not fix here:
- The contracts CHANGELOG issues a **correction against `safety-rules.json`** (G1 EC gate
  multiplier is 1.5x, not the OCR-garbled ~1.2x). Not in Phase 1 scope (reuse gate is
  process control), but per contracts rule 5 the contracts repo wins; the maintainer loop
  should reconcile `safety-rules.json` and log it.
- `instrument-tags.md` lists the **MQTT payload schema version** as "owned by the software
  side (ADR-0004); hardware must review". Phase 1 produces schema v1 — the HLD should
  name a hardware-side review of it as an output.

## Notes for downstream steps

- **Sim harness placement vs ADR-0008.** The repo layout has no home for the sim harness.
  It must not be a `backend/` package that imports `firmware/` code (rule 2: firmware and
  backend couple only via the MQTT schema). HLD decides: own top-level dir (e.g. `sim/`)
  or a virtual build target under `firmware/`. Do not split anything; just place it.
- **Fault switches scope.** The locked deliverable 4 includes fault switches (offline,
  stale, implausible), which `requirements.md` files under FR-SIM-02 (S, R2). The release
  plan's lock wins; treat those three switches as in scope and note the FR-ID drift in
  the domain brief rather than re-litigating it.
- **CI already has both jobs.** `.github/workflows/build-test-deploy.yml` has a Go
  (`go build/vet/test`) job and a Node 20 job; the "inherited Node assumptions" note in
  the release plan is largely addressed by commit 2922bc2. `build-deploy-engineer` still
  needs to uncomment/wire compose services and add the deploy step.
- **OQ-3 stays open** and gates only real room-controller firmware; nothing here waits
  on it.
