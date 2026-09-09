# Open questions, architecture risks, next actions

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** open questions (§1), architecture risks (§2), recommended next
actions (§3) of the foundation analysis. Numbered OQs are referenced from across the
docs/ set — this file is their single home.

## 1. Open questions

**Owner decisions recorded 2026-09-07** (binding on the docs set):

1. The Sep-2026 engineering set (RK-A SYS/MFG/WRS/ROOM Rev 2–3) is **draft-stage — the
   initial idea**: direction adopted, details mutable; the platform models around it
   without hardcoding its specifics (ADR-0002 absorbs revisions).
2. **Aquatic production is confirmed in the plan, sequenced after the first microgreen
   rollout** (OQ-5 disposition below) — aquatic zone engineering and aquatic protocols
   are deferred to that phase.
3. **Flood-and-drain is the confirmed irrigation choice for microgreens** (over NFT) —
   strategy-doc NFT references are superseded; Phase C product 1.3 (NFT trays) is
   pending portfolio re-evaluation.

**Owner decisions recorded 2026-09-08** (Phase 1 handover — binding on the docs set):

4. **ADR-0001…0005 are approved as written** and are now `accepted`. They are settled
   decisions: `/extend` may rely on them, and they are not re-litigated per change.
5. **Phase 1 (Release 1) scope is locked as written** in
   [release-plan.md](../roadmap/release-plan.md), **including the week-1 language spike
   gate** (OQ-16 → ADR-0006 before the bulk of the backend build).
6. **Saffron is deferred indefinitely** (OQ-1 answered — see below). No recipe work is
   required now; recipes remain an R3 concern and are unchanged by this.

All owner-facing open questions from the foundation analysis are now answered or
explicitly deferred. The remaining OQs are engineering/domain/external items, not
owner-decision items — see the table for each one's resolution path.

| # | Question | Why it matters | Blocks | Owner / resolution path |
|---|---|---|---|---|
| OQ-1 | **Is saffron still in the program?** Research treats it as a pillar (zone, product 2.3, moat); current direction (microgreens 70–80%, aquatic 20–30%, exotic optional) omits it. | Orphans Phase C product 2.3 + roadmap slots; affects floor allocation. | **ANSWERED (2026-09-08, owner): saffron is deferred indefinitely** — not in the program, no floor allocation, no recipe work required. Phase C product 2.3 and the Phase E saffron/moat weighting are orphaned as documented-but-inactive; `safety-rules.json` `saffron_corm_protocol` is **retained** (it is sourced reference data and costs nothing dormant — do not delete sourced thresholds to reflect a scope decision). Nothing is deleted from `knowledge/cea/`: the research stands, the program choice differs. | **Closed.** Re-opening is an owner decision + a fresh `/ship`; the crop model is crop-agnostic (ADR-0002/0003), so re-entry costs a recipe definition, not a redesign. |
| OQ-2 | **EC operating target (TBD in safety-rules) + water-recovery gate G1 value (OCR-unreliable).** | Any dosing/fertigation closed loop or gate coding consumes these. | R7 dosing loops; water-skid firmware gates | Domain expert + visual re-read of WRS p.8 in a vision-capable (plain-Claude) session |
| OQ-3 | **What physically is the room controller?** Engineering docs assign it a local time-series DB + local UI + bus master — beyond ESP32-class; no hardware/platform is specified anywhere. | R1–R2 firmware architecture depends on it (what runs where). | **R1 firmware design — and, by owner decision 2026-09-08, the Phase 1 build is PAUSED pending this.** | **[Decision brief ready](../pipeline/oq3-room-controller-platform/00-decision-brief.md)** (2026-09-08). Confirmed: **no source document names a platform** — this is a genuine unknown, not a lookup. Brief derives the 15 requirements from the accepted docs, disqualifies ESP32-class on R7/R8/R9/R11, and recommends **A (Linux SBC)** or **C (industrial/panel PC)** on procurement grounds. Flags that **B (collapse onto the facility host) would violate ADR-0001** by putting the room controller and backend in one failure domain — choosable only by knowingly amending that ADR. Needs: hardware program's platform/enclosure constraint, and whether "local UI" means a physical panel (→ C) or a LAN browser page (→ A). Closes as **ADR-0007**. |
| OQ-4 | **Physical-fact reconciliation**: earthing 4 vs 6 mm²; rack mass 112.7 vs 125.4 kg; LSH-04 200 vs 250 mm; UV 100 vs 108; settle/test durations; chiller size; power-table row 1; sensor counts. | These are the facts firmware/electrical work will cite. | Anything coding against them | Visual pass on source PDFs (vision-capable session) → update `safety-rules.json` citations |
| OQ-5 | **Where do aquatic plants grow?** Current room engineering (55–70% RH, flood-and-drain, microgreen trays) contains **zero aquatic accommodation**; Phase A specifies >80% RH emersed propagation with domes/misting. | The 20–30% business line has no engineered home. | **ANSWERED IN PART (2026-09-07, owner): aquatic production is confirmed, sequenced *after* the first microgreen rollout.** Where/how remains an engineering question, now deliberately deferred to the aquatic phase. | Nothing near-term; aquatic zone/facility design when that phase begins (owner + botany + mechanical) |
| OQ-6 | **Per-crop microgreen baselines** (varieties, lifecycle days, seeding density, blackout duration, expected yield/tray) — no source document has them. | R3 recipe templates need meaningful defaults; forecast confidence needs actual/expected history. | R3 templates, R4 confidence | Operator experience + own trials (the R&D loop); record provenance as operator-entered — **priority domain input while aquatic is deferred (owner, 2026-09-07)** |
| OQ-7 | **Aquatic batch lifecycle shape** — propagation from mother stock vs seed; transition-to-submersed tracking. | Generic state machine may need phase extension. | R3 (aquatic only) | botany-horticulture-specialist review before R3 design |
| OQ-8 | **What are "exotic plants"?** Undefined in every source. | Scope for a third crop category. | Nothing yet (model is open) | Owner clarification |
| OQ-9 | **Backup/offsite target + host provisioning.** FR-BKP-03 marked [A]; where do scheduled backups land off-facility, and what is the facility host? | Data-loss exposure; NFR-DAT-04 RTO/RPO realism. | R1 backup implementation | Owner + build-deploy-engineer at deploy time |
| OQ-10 | **Forecast confidence parameters** (phase bases, floors, history multiplier) — mechanism specified; all numbers are placeholders. | Overselling/under-selling risk. | R4/R5 | Domain review + operator experience before R4 ships numbers |
| OQ-11 | **Food-safety & compliance for selling microgreens in India** (FSSAI license/hygiene, testing cadence, packaging, shelf life). | Portal legality; inventory attributes (best-before); operations. | R4 inventory attributes, R5 portal launch | Owner + regulatory research (external) |
| OQ-12 | **Remote-access/portal-serving architecture.** Facility host is LAN; portal needs customer reach; VPN/relay vs cloud-publish. | Security surface (security-architecture §3) + availability. | R5 | security-safety-reviewer + build-deploy at R4/R5 boundary |
| OQ-13 | **B2B commercial mechanics** (pricing, customers, delivery radius, cold chain, cancellation policy). | Portal order flow specifics. | R5 details | Owner |
| OQ-14 | **Photoperiod actuation mechanism** — 0–10 V dim-to-off vs relay switching; depends on LED driver selection ("0-10 V or DALI", fixtures not yet chosen). | `light.schedule` on real hardware. | R2 on physical hardware (not in sim) | Electronics specialist + driver selection |
| OQ-15 | **LED fixture model/wattage** (excluded from rack BOM). | Real-energy estimates; commissioning tests. | R4 energy accuracy; R2 commissioning | Hardware program procurement |
| OQ-16 | **Backend language — Go vs Node.js/TS.** Node was inherited from the bootstrap scaffold (CI/compose detection), never weighed; owner flagged a Go preference. Evaluated 2026-09-07: performance is a non-factor at this ingest volume (single-digit msg/s at design headroom); the real differentiators are runtime/ops fit on the facility host (Go edge), API/JSON-Schema tooling breadth + full-stack TS (Node edge), possible Linux-SBC room-controller synergy (Go edge, contingent on OQ-3), hiring pool (Node edge), and **owner fluency (decisive)**. | Chooses the language before the bulk of backend build; affects only the CI job + compose build context — the architecture is language-agnostic (MQTT/TLS, Postgres/Timescale, OpenAPI boundaries all fixed). | **R1 week 1, decided by spike** (owner decision 2026-09-07): build the minimal ingest path (MQTT → plausibility gates → Timescale write → one read API endpoint + one rollup query) **both ways** against the sim harness, timeboxed to days; evaluate on owner-fluency review, facility-host fit, agent-authored-code discipline, pipeline velocity; record **ADR-0006** immediately after, before the rest of the backend build. | **ANSWERED (2026-09-08, owner): Go** — [ADR-0006](../adr/0006-go-as-the-backend-language.md). The two-arm spike was **not run**: the owner supplied the decisive criterion (fluency) at kickoff, and the [spike protocol](../pipeline/oq16-backend-language-spike/00-spike-protocol.md) §4 pre-authorised a clear fluency answer to win outright. No benchmark backs this and none was needed — throughput was ruled a non-factor before any code existed. |

## 2. Architecture risks

| # | Risk | Likelihood/Impact | Mitigation (already designed or required) |
|---|---|---|---|
| AR-1 | **Room controller = single MQTT bridge** — one choke point for platform visibility (not safety; racks are autonomous). | M/ M | Buffer + catch-up by design (ADR-0001/0004); monitor its uptime as a first-class device; consider dual-homing later. |
| AR-2 | **Room controller platform undefined** (OQ-3) — R1–R2 firmware design could be built on the wrong compute assumption. | M/H | **Being actively closed:** [decision brief](../pipeline/oq3-room-controller-platform/00-decision-brief.md) ready 2026-09-08; owner paused the Phase 1 build behind it rather than building on an assumption. Closes with ADR-0007. |
| AR-3 | **OCR-unreliable facts leak into code** (G1 gate, chiller, power row). | M/H (a bad gate value is a physical-safety event) | safety-rules.json TBD-block discipline + visual pass before coding against them (OQ-2/OQ-4); security-safety-reviewer gate. |
| AR-4 | ~~**Zero-commit repo**~~ — **CLOSED 2026-09-08.** Initial commit made; remote `trophic-corp/cea-orchestrator` configured. | — | Residual: branch protection is **not yet verified** (`gh` is unauthenticated on this host) — see next action #1. |
| AR-5 | ~~**GLM-5.3 text-only**~~ — **CLOSED 2026-09-08.** The workspace now runs on Claude (subscription OAuth); the model reads images and PDFs directly, so no separate visual-pass session is needed. | — | Consequence: next action #4 (visual re-read of the unreliable OCR zones, OQ-2/OQ-4) is now executable *in a normal session* — it changed from a scheduling constraint into ordinary work. |
| AR-6 | **Recipe-immutability discipline erodes** (convenience edit on a referenced version). | M/H | DB-level enforcement (ADR-0003), not convention; tests attempt tampering in CI. |
| AR-7 | **Forecast over-promising** — confidence floors misset; customer orders unfulfillable. | M/H | Never expose raw expected yield (working rule 16); floors + risk-adjustment are policy-configurable; abort-impact alerting; manual-override-with-reason escape hatch only. |
| AR-8 | **Alert fatigue** with 1–2 operators → real alerts dismissed. | M/M | Severity tiers, per-zone thresholds, alert-review workflow (IA §5), alert-fatigue flagging in maintenance review. |
| AR-9 | **Product-scope creep**: aquarium features leaking into CEA UI (or vice versa) because the platform is shared. | M/M | ADR-0002/0005 + persona/surface separation (IA §2); reviewer checks. |
| AR-10 | **Portal = new internet exposure** for a control-adjacent system. | M/H | Separate identity domain, no console-feature leakage, VPN/relay decision (OQ-12) behind security review; portal never in any control path. |
| AR-11 | **Per-tier sensing absence** (standard build) limits zone-level R&D resolution — T/RH is rack-level. | Certain/L-M | Capability-aware analytics (mark resolution); decide Pro-build/per-tier sensing early (OQ: hardware program) if R&D-grade per-tier analysis is wanted. |
| AR-12 | **Energy/resource estimates mistaken for measurements** in operator-facing views. | M/M | Provenance tags rendered everywhere (rd-data-model §2); estimates visually distinct. |
| AR-13 | **Bus factor 1** — single operator, single facility host, small team. | H/M | Docs-as-operating-system (this set); runbooks; maintainer-agent health scans; git everything. |
| AR-14 | **Premature ML / premature SaaS** pressure. | M/M | Explicit release gates (R8 data-quality review; ADR-0005 triggers); working rules 8/17 enforced by roadmap wording. |
| AR-15 | **India data-residency expectation** (Phase D recommends India-region) unconfirmed as requirement. | L/M | Track as compliance TODO with portal/cloud-publish decision (OQ-12). |

## 3. Recommended next actions

| # | Action | Owner | When |
|---|---|---|---|
| 1 | ~~Initial git commit + remote~~ **DONE 2026-09-08.** Remaining: **authenticate `gh` on the facility/dev host and verify branch protection** on `trophic-corp/cea-orchestrator` — `gh auth status` reports no login, so `github-ops-agent` cannot open PRs or set protection. Note the remote org is `trophic-corp`, while CLAUDE.md says `trophic`; reconcile the two. | Owner (`gh auth login`), then github-ops-agent | **Blocks the first Phase 1 PR** (not the build) |
| 2 | ~~Owner decisions on ADR-0001…0005~~ — **DONE 2026-09-08: approved as written, all five now `accepted`.** | Owner | Complete |
| 3 | ~~Resolve OQ-1 (saffron) and OQ-5 (aquatic)~~ — **DONE.** OQ-5 sequenced 2026-09-07; **OQ-1 closed 2026-09-08 (saffron deferred indefinitely).** | Owner | Complete |
| 4 | **Visual pass on the 4 unreliable OCR zones + physical-fact register (OQ-2/OQ-4)**; update safety-rules.json citations afterward. **No longer needs a special session** — the workspace model reads PDFs directly (AR-5 closed). | Any normal session + domain expert for the EC target | Before any firmware coded against gates/dimensions. **Not a Phase 1 blocker** — Phase 1 has zero actuation, so no gate value is consumed. |
| 5 | **Decide the room controller compute platform** (OQ-3) — it shapes R1–R2 firmware. **This is the one unresolved gate sitting inside locked Phase 1 scope.** It does *not* block the week-1 spike or the sim-backed build (the room controller is a virtual device in sim, and the rack controller is ESP32-class regardless), but it blocks *real* room-controller firmware and it feeds the OQ-16 language decision (a Linux-SBC room controller is a point in Go's favour). | Hardware program + iot-control-systems-engineer — **[brief ready for decision](../pipeline/oq3-room-controller-platform/00-decision-brief.md)** | **NOW — this is the critical path.** Owner decision 2026-09-08: Phase 1 build pauses until OQ-3 closes. |
| 6 | **Wire the two new agents into pipeline commands** (`/ship`/`/extend`: iot-control-systems at HLD for control-plane kinds; ux-product-designer before LLD for operator-facing kinds) + fix CLAUDE.md/README agent-count and aspirational wording. | Maintainer/owner via a small `/extend docs-only` run | With R1 kickoff |
| 7 | **Confirm hardware program timeline** (rack controller boards, room controller, first rack availability) to set R1/R2 validation dates; sim contract means software proceeds regardless. | Hardware program | R1 kickoff |
| 8 | **Begin capturing microgreen baselines** (OQ-6) from operator experience — even rough starting numbers with operator-entered provenance beat templates with invented numbers. | Sholaverde operator + botany specialist | Before R3 |
| 9 | ~~Phase 1 (Release 1) kickoff~~ — **STARTED 2026-09-08** on owner go-ahead. Decisions recorded (ADRs accepted, OQ-1 closed, **OQ-16 → Go, ADR-0006**). **Build is PAUSED at owner's direction pending OQ-3** (action #5), which is now the critical path. Resumes with the Phase 1 `/ship` once ADR-0007 lands. | Pipeline | Paused on #5 |

## 4. Standing separation (restated for every future doc/PR)

Requirement → in `docs/requirements/`. Assumption → marked **[A]** inline + listed here as
an OQ. Recommendation → advisory language, changeable. Decision → ADR. Anything physical
and quantitative → sourced from `safety-rules.json` / `knowledge/cea/`, or it's a flagged
unknown. Never guess a safety threshold; never present forecast as stock; never edit a
referenced recipe version.