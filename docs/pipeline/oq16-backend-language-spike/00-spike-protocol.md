# OQ-16 backend language spike — protocol & scorecard

**Status:** **CLOSED 2026-09-08 — spike not run; collapsed on the decisive criterion.**
The owner supplied criterion 1 (fluency: **Go**) at kickoff, before either arm was built.
§4 states that a clear owner-fluency answer wins outright, so building a Node arm would
have gathered evidence for a settled question. Outcome recorded in
**[ADR-0006](../../adr/0006-go-as-the-backend-language.md)**.
This document is retained as the record of *how the decision would have been scored* —
it is the reason the collapse is legible rather than a shortcut taken quietly.
· **Date:** 2026-09-08
**Decides:** [OQ-16](../../decisions/open-questions-risks-next-actions.md) — Go vs Node.js/TS
**Produces:** ADR-0006, before the bulk of the Phase 1 backend build
**Gate:** Phase 1 week-1, per the locked [release plan](../../roadmap/release-plan.md)

> This protocol is written **before either arm exists**, deliberately. A spike scored
> after the code is written scores the author's enthusiasm, not the languages. The
> scorecard in §4 is fixed as of this document; changing a weight after seeing results
> requires saying so explicitly in ADR-0006.

## 1. What is being decided — and what is not

**Decided:** the implementation language for the Phase 1 backend service (registry,
ingest, alerts, backup, read API).

**Not decided, and not re-openable by this spike** — these are settled by accepted ADRs
and the spike must not disturb them:

| Fixed by | What is fixed |
|---|---|
| ADR-0004 | MQTT (Mosquitto) over TLS; PostgreSQL + TimescaleDB as the single store |
| ADR-0002 | Capability-based device model — no hardcoded device types |
| ADR-0005 | `org_id` on every core entity; every query path org-scoped |
| ADR-0001 | Cloud/backend is never in the actuation path |
| device-control-model §4 | Topic taxonomy and QoS/retain semantics |

The architecture is language-agnostic by design. A wrong answer here costs a rewrite of
one service, not a redesign — which is exactly why it is worth a two-day spike and not a
two-week argument.

## 2. The slice each arm builds

Identical scope, identical acceptance, both arms. From OQ-16: *MQTT → plausibility gates
→ Timescale write → one read API endpoint + one rollup query.*

1. **Subscribe** to `trophic/<org>/<site>/<room|rack>/<tier?>/<device>/telemetry/<metric>`
   at QoS 1, over TLS.
2. **Parse** the sample envelope: `ts_source` (device clock), `seq` (per-device counter),
   value, `calibration_id`.
3. **Apply plausibility gates** (device-control-model §6) and assign
   `quality ∈ {good, suspect, implausible-rejected}`:
   - **Range** — against the *sensor's documented physical range*, e.g. RH 0–100 %,
     T −40..85 °C, CO2 0–10 000 ppm.
   - **Rate-of-change** — faster than the sensor can physically move → `suspect`.
   - **Stuck-value** — N identical samples beyond the noise floor → `suspect`.
   - **Verdicts flag, never drop** (FR-TEL-02). A rejected sample is still stored, marked.

   > **Design rule the spike must honour, and the easiest thing to get wrong:**
   > plausibility gates use **sensor physical ranges, not crop target bands**. The crop
   > bands in `safety-rules.json` (PPFD 150–210, RH 55–70 %, VPD 0.6–1.0 kPa) are
   > *control targets*. A real 45 % RH reading is a **true reading of a bad condition** —
   > an alert. It is not an implausible sample. An arm that conflates the two fails
   > acceptance regardless of how good its code looks.

4. **Write** to TimescaleDB, hypertable keyed `(org_id, zone, device, metric, ts)`
   (FR-TEL-01), `org_id` non-null and enforced (ADR-0005).
5. **Read API** — one endpoint: latest-per-metric for a zone, returning value, quality,
   and **data age** (FR-TEL-04 — age is never optional).
6. **Rollup query** — one continuous aggregate, 1-minute mean per (device, metric),
   quality-filtered (FR-TEL-05).

**Explicitly out of the spike:** auth, the console, alerting, backup, commands/acks
(Phase 1 has no actuation at all), capability discovery. Those are the *build*, not the
*decision*.

## 3. Shared harness — built once, used by both arms

Neither arm gets a bespoke test rig; that is how spikes get rigged. One publisher, one
compose file, both arms plugged into it in turn.

- **Virtual device publisher** (sim harness v0 seed, FR-SIM-01) — one virtual rack, 4
  tiers, publishing T/RH/CO2 at realistic cadence, with **fault switches**: offline,
  stale, implausible-high, stuck-value, clock-skew (`ts_source` in the past/future),
  duplicate `seq` (broker redelivery), out-of-order arrival.
- **Mosquitto + TimescaleDB** from the Phase 1 compose stack — same versions both arms.
- **Fixed replay corpus** — one recorded stream, byte-identical for both arms, so gate
  verdicts are diffable sample-by-sample.

**Acceptance (both arms must pass before scoring — an arm that fails is not scored, it
is fixed or reported as a finding):**
- 72-hour-equivalent replay with **zero unaccounted gaps** — a gap must be a *recorded*
  gap (the Phase 1 validation criterion, applied early).
- Duplicate `seq` and broker redelivery produce **one** stored sample, not two.
- Out-of-order and clock-skewed samples land correctly, `ts_source` preserved distinctly
  from ingest time.
- Every gate verdict matches the reference table, sample for sample, across both arms.
  **Where the two arms disagree, the reference table is wrong or ambiguous** — that is a
  finding worth more than the language decision itself, and it gets written down.
- Broker restart mid-stream → reconnect, resubscribe, no loss, no duplication.

## 4. Scorecard — weights fixed 2026-09-08

Per OQ-16, **owner fluency is decisive**; the rest inform it. Weights are stated so the
write-up cannot quietly re-rank them.

| # | Criterion | Weight | How it is judged — evidence, not vibes |
|---|---|---|---|
| 1 | **Owner fluency** | **Decisive** | Owner reads both arms cold and says which one they could debug at 2 a.m. during a grow. Single most important input; a clear owner preference **wins outright** even if 2–5 lean the other way — this is a small team with a bus factor of 1 (AR-13). |
| 2 | Facility-host ops fit | High | Deploy footprint, cold-start, memory under sustained ingest, single-binary vs runtime+`node_modules`, behaviour on an unattended box that must survive power events. Measured on the compose stack, not asserted. |
| 3 | Agent-authored-code discipline | High | Which arm's failure modes are caught by the compiler/types rather than at runtime? Most code here will be agent-authored; the language that makes a bad edit *fail loudly* is worth more than the one that makes a good edit shorter. |
| 4 | Pipeline velocity | Medium | Wall-clock to working arm, and honest friction notes from building it. |
| 5 | Ecosystem fit | Medium | MQTT client maturity, Timescale/pg driver, OpenAPI/JSON-Schema tooling. Both are adequate here — this criterion mostly cannot separate them, and the write-up should say so if it doesn't. |
| 6 | Room-controller synergy | **Conditional — [OQ-3]** | If the room controller is a Linux SBC, sharing a language with it is a real Go advantage. **[OQ-3] is unresolved.** If it is still open at scoring time, this criterion is recorded as *indeterminate* and given **zero weight** — not guessed. |
| — | Raw throughput | **Zero** | Already settled in OQ-16: single-digit msg/s at design headroom. Both arms will be orders of magnitude clear. **Any benchmark either arm wins is noise, and citing it in ADR-0006 is a scoring error.** |

## 5. Sequence

1. Shared harness (publisher + compose + replay corpus + reference gate table).
2. Both arms, against the harness. Build order is recorded — the second arm benefits from
   lessons learned, and the write-up must note this rather than pretend symmetry.
3. Both arms pass §3 acceptance, or the failure is reported.
4. Owner reads both (criterion 1).
5. **ADR-0006** records the decision, the scorecard as actually applied, and the losing
   arm's fate (deleted, or kept as a reference implementation).
6. The losing arm's code is **removed from the build**, not left to rot as a
   half-maintained second implementation.

## 6. Open dependencies

| Item | Effect | Blocking? |
|---|---|---|
| [OQ-3] room controller platform | Criterion 6 | **No** — scored as indeterminate/zero-weight if unresolved. Resolving it first makes the spike sharper. |
| `gh` unauthenticated on this host | No PR can be opened for the spike branch | **Not for building** — blocks landing. See next-action #1. |
| [OQ-2]/[OQ-4] unreliable OCR values | None | **No** — the spike touches no safety threshold; plausibility gates use sensor datasheet ranges, and Phase 1 has zero actuation. |
