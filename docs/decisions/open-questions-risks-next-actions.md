# Open questions, architecture risks, next actions

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** open questions (§1), architecture risks (§2), recommended next
actions (§3) of the foundation analysis. Numbered OQs are referenced from across the
docs/ set — this file is their single home.

## 1. Open questions

| # | Question | Why it matters | Blocks | Owner / resolution path |
|---|---|---|---|---|
| OQ-1 | **Is saffron still in the program?** Research treats it as a pillar (zone, product 2.3, moat); current direction (microgreens 70–80%, aquatic 20–30%, exotic optional) omits it. | Orphans Phase C product 2.3 + roadmap slots; affects floor allocation. | R3 recipe scope (minor — model is crop-agnostic) | Sholaverde/Trophic owner decision |
| OQ-2 | **EC operating target (TBD in safety-rules) + water-recovery gate G1 value (OCR-unreliable).** | Any dosing/fertigation closed loop or gate coding consumes these. | R7 dosing loops; water-skid firmware gates | Domain expert + visual re-read of WRS p.8 in a vision-capable (plain-Claude) session |
| OQ-3 | **What physically is the room controller?** Engineering docs assign it a local time-series DB + local UI + bus master — beyond ESP32-class; no hardware/platform is specified anywhere. | R1–R2 firmware architecture depends on it (what runs where). | R1 firmware design | Hardware program + iot-control-systems-engineer; likely an SBC/industrial host or the facility host itself — must be decided before R1 |
| OQ-4 | **Physical-fact reconciliation**: earthing 4 vs 6 mm²; rack mass 112.7 vs 125.4 kg; LSH-04 200 vs 250 mm; UV 100 vs 108; settle/test durations; chiller size; power-table row 1; sensor counts. | These are the facts firmware/electrical work will cite. | Anything coding against them | Visual pass on source PDFs (vision-capable session) → update `safety-rules.json` citations |
| OQ-5 | **Where do aquatic plants grow?** Current room engineering (55–70% RH, flood-and-drain, microgreen trays) contains **zero aquatic accommodation**; Phase A specifies >80% RH emersed propagation with domes/misting. | The 20–30% business line has no engineered home. | R3 aquatic batch modeling; facility planning | Owner + botany + mechanical specialists — needs a facility decision (dedicated rack? zone retrofit? later phase?) |
| OQ-6 | **Per-crop microgreen baselines** (varieties, lifecycle days, seeding density, blackout duration, expected yield/tray) — no source document has them. | R3 recipe templates need meaningful defaults; forecast confidence needs actual/expected history. | R3 templates, R4 confidence | Operator experience + own trials (the R&D loop); record provenance as operator-entered |
| OQ-7 | **Aquatic batch lifecycle shape** — propagation from mother stock vs seed; transition-to-submersed tracking. | Generic state machine may need phase extension. | R3 (aquatic only) | botany-horticulture-specialist review before R3 design |
| OQ-8 | **What are "exotic plants"?** Undefined in every source. | Scope for a third crop category. | Nothing yet (model is open) | Owner clarification |
| OQ-9 | **Backup/offsite target + host provisioning.** FR-BKP-03 marked [A]; where do scheduled backups land off-facility, and what is the facility host? | Data-loss exposure; NFR-DAT-04 RTO/RPO realism. | R1 backup implementation | Owner + build-deploy-engineer at deploy time |
| OQ-10 | **Forecast confidence parameters** (phase bases, floors, history multiplier) — mechanism specified; all numbers are placeholders. | Overselling/under-selling risk. | R4/R5 | Domain review + operator experience before R4 ships numbers |
| OQ-11 | **Food-safety & compliance for selling microgreens in India** (FSSAI license/hygiene, testing cadence, packaging, shelf life). | Portal legality; inventory attributes (best-before); operations. | R4 inventory attributes, R5 portal launch | Owner + regulatory research (external) |
| OQ-12 | **Remote-access/portal-serving architecture.** Facility host is LAN; portal needs customer reach; VPN/relay vs cloud-publish. | Security surface (security-architecture §3) + availability. | R5 | security-safety-reviewer + build-deploy at R4/R5 boundary |
| OQ-13 | **B2B commercial mechanics** (pricing, customers, delivery radius, cold chain, cancellation policy). | Portal order flow specifics. | R5 details | Owner |
| OQ-14 | **Photoperiod actuation mechanism** — 0–10 V dim-to-off vs relay switching; depends on LED driver selection ("0-10 V or DALI", fixtures not yet chosen). | `light.schedule` on real hardware. | R2 on physical hardware (not in sim) | Electronics specialist + driver selection |
| OQ-15 | **LED fixture model/wattage** (excluded from rack BOM). | Real-energy estimates; commissioning tests. | R4 energy accuracy; R2 commissioning | Hardware program procurement |

## 2. Architecture risks

| # | Risk | Likelihood/Impact | Mitigation (already designed or required) |
|---|---|---|---|
| AR-1 | **Room controller = single MQTT bridge** — one choke point for platform visibility (not safety; racks are autonomous). | M/ M | Buffer + catch-up by design (ADR-0001/0004); monitor its uptime as a first-class device; consider dual-homing later. |
| AR-2 | **Room controller platform undefined** (OQ-3) — R1–R2 firmware design could be built on the wrong compute assumption. | M/H | Decide OQ-3 before R1 firmware /ship; sim contract keeps platform software decoupled meanwhile. |
| AR-3 | **OCR-unreliable facts leak into code** (G1 gate, chiller, power row). | M/H (a bad gate value is a physical-safety event) | safety-rules.json TBD-block discipline + visual pass before coding against them (OQ-2/OQ-4); security-safety-reviewer gate. |
| AR-4 | **Zero-commit repo** — all work is untracked; no CI, no review trail, one `rm -rf` from catastrophe. | H/M | Next action #1: initial commit + remote + branch protection. |
| AR-5 | **GLM-5.3 text-only** — the visual verification this program needs (engineering drawings, OCR disputes) cannot be done by the default agents. | Certain/M | Keep `claude-anthropic` visual-pass sessions in the workflow explicitly (CLAUDE.md already documents this). |
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
| 1 | **Initial git commit + remote + branch protection** under the `trophic` org — everything currently rides on an untracked working tree. | github-ops-agent (with user confirmation) | Immediately, on approval of this analysis |
| 2 | **Owner decisions on ADR-0001…0005** (approve/amend each; ADR-0003's DB-level immutability and ADR-0005's org-scoping are the two with lasting cost if wrong). | Owner | Before any implementation |
| 3 | **Resolve OQ-1 (saffron) and OQ-5 (aquatic home in room design)** — the two business-shape questions that gate R3 design and facility planning. | Owner + specialists | Before R3 design starts |
| 4 | **Vision-capable (plain-Claude) session** on the 4 unreliable OCR zones + physical-fact register (OQ-2/OQ-4); update safety-rules.json citations afterward. | A session with image capability (GLM cannot) | Before any firmware coded against gates/dimensions |
| 5 | **Decide the room controller compute platform** (OQ-3) — it shapes R1–R2 firmware. | Hardware program + iot-control-systems-engineer | Before R1 firmware /ship |
| 6 | **Wire the two new agents into pipeline commands** (`/ship`/`/extend`: iot-control-systems at HLD for control-plane kinds; ux-product-designer before LLD for operator-facing kinds) + fix CLAUDE.md/README agent-count and aspirational wording. | Maintainer/owner via a small `/extend docs-only` run | With R1 kickoff |
| 7 | **Confirm hardware program timeline** (rack controller boards, room controller, first rack availability) to set R1/R2 validation dates; sim contract means software proceeds regardless. | Hardware program | R1 kickoff |
| 8 | **Begin capturing microgreen baselines** (OQ-6) from operator experience — even rough starting numbers with operator-entered provenance beat templates with invented numbers. | Sholaverde operator + botany specialist | Before R3 |
| 9 | **Phase 1 (Release 1) kickoff** via `/ship` on approval of this foundation + the roadmap's Phase 1 scope. | Pipeline | On approval |

## 4. Standing separation (restated for every future doc/PR)

Requirement → in `docs/requirements/`. Assumption → marked **[A]** inline + listed here as
an OQ. Recommendation → advisory language, changeable. Decision → ADR. Anything physical
and quantitative → sourced from `safety-rules.json` / `knowledge/cea/`, or it's a flagged
unknown. Never guess a safety threshold; never present forecast as stock; never edit a
referenced recipe version.