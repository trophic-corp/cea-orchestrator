# Workspace & knowledge audit — 2026-09-07

**Status:** completed audit · Part of the foundation analysis (deliverables 1–5).
Findings feed [open-questions-risks-next-actions.md](../decisions/open-questions-risks-next-actions.md).
Method: full read of workspace files (agents, commands, settings, CI, compose,
safety-rules, ADR/pipeline/site READMEs) + full extraction of all `knowledge/cea/`
documents (5 phase reports + 4 OCR'd engineering docs, read in full by dedicated
extraction passes).

---

## 1. Repository state (headline findings)

- **Zero git commits.** Everything — knowledge, agents, CI, docs — is untracked. No
  history, no CI runs, no branch protection evidence, and no remote configured
  (`git remote -v` empty). The ADR process (proposed → merged → accepted) and the
  build-test-deploy workflow cannot actually function yet. **First next action.**
- `backend/`, `frontend/`, `firmware/`, `hardware/` are **empty** (`.gitkeep` only).
- `docs/adr/` has a template + README, **no ADRs** — CLAUDE.md's description of "accepted
  architectural decisions" is currently aspirational. (This audit adds ADR-0001…0005,
  status `proposed`.)
- `docs/site/` honestly marked "not yet scaffolded". CI deploy step is an honest
  placeholder. `docker-compose.prod.yml` is a fully-commented skeleton whose comments
  encode real intended decisions (Mosquitto TLS :8883 local + separate optional cloud
  client; TimescaleDB pg16; Node services; docker secrets).
- CLAUDE.md/README say "21 specialist agents" — now 23 after this audit's additions (§2).

## 2. Existing workspace/agent audit

### 2.1 What exists (21 agents at audit time)

**Pipeline/governance:** architecture-reviewer (classifier/gate), hld-architect,
lld-architect, systems-integration-reviewer (cross-reference/reconciliation),
security-safety-reviewer (combined cyber+physical, BLOCK gate, no Edit),
qa-e2e-validator (acceptance, BLOCK, no Edit), maintainer-agent (scheduled, issues-only,
no Edit), release-manager (semver/CHANGELOG), github-ops-agent (only repo/PR/secrets
actor, never merges), docs-writer (Starlight + ADR status), build-deploy-engineer
(workflows + compose).

**Domain council:** botany-horticulture-specialist (crop physiology: VPD/DLI/photoperiod,
dosing, saffron, aquatic tolerances), electronics-hardware-specialist (feasibility,
sensor/actuator precision), mechanical-industrial-design-specialist (rack/room/ergonomics),
manufacturing-supply-chain-specialist (make-vs-buy, Coimbatore sourcing).

**Implementation:** firmware-engineer (ESP-IDF), backend-engineer, frontend-engineer,
pcb-layout-engineer (KiCad), automation-engineer (test harnesses + HIL).

**Scoping:** product-strategy-specialist (PRD, portfolio/roadmap fit).

**Commands:** `/ship` (14-step full pipeline), `/extend` (lighter, ADR-gated),
`/fix` (root-cause), `/health-check` (non-blocking, issues-only). Guardrails in
`.claude/settings.json` (ask-list for force-push/reset/branch-delete/secrets/repo-edit/
workflow-locks) — sound and minimal. Model config routes GLM-5.3 text-only via
OpenRouter; vision-dependent tasks are explicitly delegated to a plain-Claude session.

**Assessment:** the orchestration design is genuinely good — risk-sized pipelines,
domain knowledge upstream of code, a never-skippable safety gate with no Edit tool,
artifact trails, ADR-reuse. Its weakness is not shape but **absence of content**: no ADRs,
no code, no runs yet.

### 2.2 Evaluation against the required specialist list

| Required role (brief) | Verdict | Detail |
|---|---|---|
| CEA domain architect | **Partial — adequate for now** | Crop physiology: botany specialist. Design: hld-architect. Contradiction-catching: systems-integration-reviewer. No single "CEA production domain" owner, but production-model decisions landed fine in this foundation pass. |
| IoT/control-systems engineer | **Was missing — CREATED** | firmware-engineer implements; electronics-hardware does feasibility; nobody owned control architecture (state machines, command semantics, offline policy, capability model). New: `iot-control-systems-engineer`. |
| Agronomy/crop-science | **Exists** | botany-horticulture-specialist covers microgreens, aquatic, saffron; instructed to flag gaps rather than invent. |
| Product manager | **Exists** | product-strategy-specialist. |
| UX/product designer | **Was missing — CREATED** | Brief demands IA/flows/wireframes before implementation; no agent owned UX. New: `ux-product-designer` (no Edit-type tools; repo = source of truth, Figma visual-only). |
| Data architect | **Partial — defer with trigger** | lld-architect owns per-feature schema; backend-engineer owns pipelines. Cross-release data/analytics strategy has no owner. **Trigger: create before R4/R6 design work.** |
| R&D/data analytics specialist | **Missing — defer** | **Trigger: create at R6 start** (analytics release); data model (rd-data-model.md) pre-supplies their needs. |
| Inventory/order-management specialist | **Missing — defer** | **Trigger: create before R4/R5 design** (lots/forecast/orders semantics). |
| Systems architect | **Exists** | hld-architect + architecture-reviewer. |
| Security architect | **Exists** | security-safety-reviewer (deliberately combined cyber + physical — correct for this domain). |
| DevOps/platform engineer | **Exists** | build-deploy-engineer + maintainer-agent. |
| QA/test engineer | **Exists** | qa-e2e-validator + automation-engineer (HIL). |
| Observability engineer | **Partial — defer** | Platform self-observability is NFR-OPE-02; build-deploy + maintainer cover initial needs. **Trigger: multi-facility or first serious incident review.** |
| Simulation/test-env engineer | **Partial — now covered** | automation-engineer (HIL) + the new iot-control-systems-engineer owns the sim contract (device-control-model §10). |
| Technical writer | **Exists** | docs-writer. |

**Skills:** the four pipeline commands are the workspace's skills; domain knowledge lives
in agents + `knowledge/cea/`. No additional skills needed now. **Recommended follow-up:**
wire the two new agents into `/ship`-family command steps (iot-control-systems at HLD for
control-plane changes; ux-product-designer before LLD for operator-facing changes) — a
small edit to `.claude/commands/*.md`, proposed as a next action rather than done
unilaterally.

## 3. Documentation audit

### 3.1 Classification (authoritative → outdated)

> **Owner decision 2026-09-07:** the Sep-2026 engineering set is **draft-stage — the
> initial idea** (direction adopted, details mutable). It remains the best available
> hardware truth and the platform builds around it, but specific parameters/topology may
> be revised by the hardware program — the platform's capability model absorbs revisions
> rather than hardcoding them.

| Document | Status | Notes |
|---|---|---|
| `knowledge/cea/cad/Rack_A_MANUFACTURING_PACK.ocr.md` (MFG Rev 2) | **Authoritative-draft** (current build definition) | draft-stage; OCR caveats per §3.3-8 |
| `…/Closed-Loop_Water_Recovery.ocr.md` (WRS Rev 3) | **Authoritative-draft** (current water architecture) | gate G1/G2 values OCR-unreliable |
| `…/CEA_Room_-_Floor_Plan.ocr.md` (ROOM Rev 3) | **Authoritative-draft** (current room layout) | power table row 1 garbled |
| `…/Rack_A_Specification.ocr.md` (SYS Rev 2) | **Partially superseded** (self-declared banner) | rack structure/irrigation valid; drain-to-waste → closed loop; header size superseded by Rev 3 pair |
| `safety-rules.json` | **Authoritative** for thresholds | sourced, citation-backed, TBDs honestly marked (2026-09-05) |
| `knowledge/cea/README.md` | **Authoritative index** | honest per-page OCR quality map |
| Phase A (literature review) | **Split validity** | Crop-physiology sections (PPFD, photoperiod, CO2, fertigation, saffron, aquatic cultivation) remain the physiological evidence base. **Facility design §2 is superseded** by the Sep-2026 engineering set (room size, zones, NFT, 2–3 tier racks, curtain partitions). |
| Phase B (benchmark) | **Current as market context** | several figures unsourced/questionable (§3.3) |
| Phase C (portfolio) | **Current with caveats** | product 1.3 (NFT trays) inconsistent with current flood-and-drain build; 2.6 bundle composition oddity |
| Phase D (BOM) | **Current as sourcing strategy** | component specs (SCD41, SHT-class, Atlas/DFRobot pH/EC); no prices/quantities (MOQs only) |
| Phase E (roadmap/TRL) | **Current with caveats** | saffron weighting diverges from current direction; SMT outsource-vs-evaluate tension |
| CLAUDE.md / README.md | **Accurate as intent, aspirational as state** | ADR corpus, prod stack, "21 agents" (now 23) |
| docs/pipeline/README, docs/adr/README, docs/site/README, CHANGELOG | Current | honest about not-yet-built things |

### 3.2 Decisions already made (documented — the platform must respect, not re-litigate)

**Owner decisions recorded 2026-09-07:**
- The Sep-2026 engineering set is **draft-stage — the initial idea**: these physical
  decisions bind the platform's *model* (it is what we build around), but the hardware
  program may revise specifics.
- **Flood-and-drain is the confirmed irrigation choice for microgreens** (owner: better
  suited than NFT) — all strategy-doc NFT references are superseded; Phase C product
  1.3 (NFT trays) is pending portfolio re-evaluation.
- **Aquatic production is confirmed in the plan, sequenced after the first microgreen
  rollout** — aquatic engineering questions are deliberately deferred, not blocking.

**Physical/control (engineering set — draft-stage initial design):**
1. Three-layer edge control: rack controllers (tier sequences, fan PWM, local
   interlocks, last-known-good) → room controller (drain token, 8-gate reuse, dosing,
   HVAC/CO2 setpoints, local TSDB + local UI) → cloud ("remote view and history,
   nothing the room depends on"). Hardwired interlocks bus-independent.
2. Field bus: Modbus RTU over RS485, wired-first; 14 nodes; room controller = bus master.
3. Per-tier flood-and-drain (NC fill / NO drain), 22 mm flood / 10 min dwell; per-tier
   0–10 V LED dimming; per-tier EC fan PWM + tacho; **climate & CO2 control is
   room-level by explicit design** (per-tier CO2 explicitly rejected).
4. Flood-and-drain replaces NFT (SYS banner + Phase C product-1.3 mismatch noted).
5. Gravity-fed terrace water system (TK-01 500 L), closed-loop recovery, 8-gate reuse,
   sequential draining mandatory, blowdown 8% daily 02:00.
6. Electrical architecture: Type A RCBOs, ELV boundary (≤ canopy = 24/48 V), room-level
   E-stop per aisle, per-rack isolation (valve+union / isolator+RCBO / bus drop).
7. Rack A form factor: 4 tiers (3–5 supported), 16× 1020 trays, 1456×690×1960 mm
   installed; 11-rack 20×12 ft room, 3 rows, 794 mm aisles.
8. Costs on record: rack Grow ₹49,296 / Pro ₹56,996 (excl. LED fixtures); plenum
   retrofit ₹2,640/rack; skid ₹79k/₹153k tiers.

**Platform/infra (scaffolding + research):**
9. MQTT over TLS to an India-region endpoint (Phase D: data residency) + local broker
   in prod stack (compose); TimescaleDB pg16; Node backend — **scaffold assumption, not
   a weighed decision; superseded 2026-09-07 by the OQ-16 R1 spike (Go vs Node.js/TS)**;
   TS frontend; ESP-IDF firmware (CI detection logic).
10. Monorepo; `/ship` `/extend` `/fix` `/health-check` pipelines; docs/pipeline artifacts;
    ADR process; github-ops-only PRs under `trophic` org; no auto-merge.
11. Phase-1 CI/CD = single prod stack; integration stack deliberately deferred.
12. Manufacturing strategy (Phase E): fabrication permanently outsourced (Coimbatore);
    firmware + substrate chemistry + Ooty performance dataset = never-outsource IP;
    imports = diodes, lab-grade sensors, rockwool, glassware.
13. Positioning: "India's first vertically-engineered mid-tech automation-and-lighting
    brand — the Priva/Ridder/ADA role" with aquascaping equipment as the identified
    whitespace (no Indian ADA-equivalent exists).

### 3.3 Contradictions & questionable claims (flagged, with dispositions)

1. **Crop mix vs strategy layer** — research allocates microgreens 30–40%, aquatic
   30–40%, saffron 10–20% (and Phase E treats saffron as a pillar/moat); current
   direction is microgreens 70–80%, aquatic 20–30%, exotic optional, saffron unmentioned.
   → [OQ-1]; platform is crop-agnostic by design, so no architecture cost either way.
2. **Irrigation subsystem** — Phase A/C specify NFT; engineering set specifies
   flood-and-drain per tier. → Resolved: engineering set is newer and is what's being
   manufactured. Phase C product 1.3 (NFT trays) needs re-evaluation against the
   flood-and-drain line (and against the brief's Trophic scope which *includes*
   flood-and-drain equipment). **Owner-confirmed 2026-09-07: flood-and-drain is the
   decision** — NFT superseded everywhere.
3. **Control granularity** — Phase A zone-level (2–3 tier racks, curtain partitions);
   engineering set per-tier. → Resolved per-tier (engineering).
4. **Aquatic zone missing from engineering** — Phase A specifies aquatic emersed
   propagation at >80% RH with humidity domes/misting; the Sep-2026 room design runs
   55–70% RH with flood-and-drain trays and **contains no aquatic accommodation at all**.
   The 20–30% aquatic business line currently has no engineered home. → **Owner
   sequencing decision 2026-09-07: aquatic production is confirmed, deliberately after
   the first microgreen rollout** — the engineering question is deferred to the aquatic
   phase, not a near-term blocker [OQ-5].
5. **CO2 "18–40%+ yield gains"** — both subsystem tables cite this; underlying §1.3 text
   supports ~18% at 550–650 ppm and qualitative gains above. The 40%+ upper bound is
   unsubstantiated. → Do not reuse without verification.
6. **Phase C 2.6 bundle** includes aquascaping starter (1.5, a hobby product) inside a
   CEA room package; unexplained. → Flag for portfolio review.
7. **Phase E SMT tension** — "permanently outsource SMT" vs Year-3 "evaluate in-house SMT
   line". → Portfolio decision, no platform impact.
8. **Small numeric conflicts in engineering set** (documented, unresolved): earthing
   conductor 4 vs 6 mm²; rack mass 112.7 vs 125.4 kg; LSH-04 height 200 vs 250 mm;
   UV dose 100 vs 108 mJ/cm²; settle/test durations; sensor-count headline garbled;
   chiller capacity garbled; floor-plan power row 1 garbled. → Physical-fact register
   needed; visual (vision-capable) pass on the source PDFs before firmware/coding
   against any of them.
9. **Reference quality** — Phase B figures sourced from Wikipedia/trade press/market
   reports (Netafim share, Spread output, "AeroFarms entering India" unsourced and
   dubious); Phase A has authorless citations; LED efficacy claims (4.0 µmol/J) above
   commonly reported best-in-class. → Usable as context; not load-bearing for the
   platform; re-verify before external/marketing use.
10. **Self-obsolete market absolutes** — "no Indian premium aquascaping brand" predates
    Trophic's own aquarium line plans (per the brief). → Treat Phase B whitespace claims
    as *opportunity* statements, not current-state facts.
11. **Workspace-state drift** — CLAUDE.md/README describe ADR corpus + deployed prod
    stack as existing; both are scaffolding-intent only. → Fix wording at next edit;
    initial commit will start making them true.

## 4. CEA knowledge audit

- **Coverage:** crop physiology for microgreens (strong: spectra, PPFD 150–210, 16–24 h
  photoperiod, 24 h-continuous tradeoff), saffron (two-phase protocol, sourced in
  safety-rules), aquatic (emersed propagation practice; evidence honestly labeled
  trade/hobbyist-grade — reframed as an IP opportunity: publishing own trial data).
  Engineering knowledge for rack/room/water/electrical is excellent and current.
- **Gaps within coverage:** no per-crop lifecycle/yield baselines (radish vs pea vs
  basil…), no numeric VPD target in Phase A (floor plan supplies 0.6–1.0 kPa), no
  DLI (derivable, flagged), no aquatic-room engineering (see 3.3-4), no food-safety
  literature at all.
- **OCR reliability (from knowledge/cea/README + extraction):** prose/tables good; four
  known-unreliable zones — Rack A SYS pages 10–11 airflow schematic (scrambled),
  floor-plan drawing pages 2–3 (only setting-out table partially usable), WRS page 8
  gate G1 (internally inconsistent), SYS page 5 GA dimensions. These need a
  vision-capable (plain-Claude) session; GLM-5.3 cannot do this pass. Plus the §3.3-8
  register from this audit.
- **safety-rules.json:** high quality, citation-backed; hard TBDs: **EC operating
  target**, **EC recovery gate G1** (OCR-unreliable) — both blocking for R7 dosing
  logic; DLI derivable-not-sourced (acceptable); open structural items (depth-plane
  bracing T16 test; canopy-velocity CV commissioning measurement; terrace slab
  structural sign-off before tank order).

## 5. Missing domain knowledge (blocking analysis per item)

**Platform-blocking (must resolve before the named release):**
- Aquatic production engineering (where/how in the room; humidity, trays, protocols) →
  **deferred by owner decision 2026-09-07** (aquatic starts after the first microgreen
  rollout); re-opens with the aquatic phase [OQ-5].
- Per-crop microgreen baselines (lifecycle days, seeding density, expected yield/tray,
  blackout phases) → blocks R3 recipe templates having meaningful defaults; owner:
  Sholaverde operator experience + own trials (the R&D loop itself) [OQ-6].
- EC operating target + G1 gate value → block R7 closed-loop fertigation [OQ-2].
- Forecast confidence curve parameters → blocks R4/R5 customer-visible availability
  (mechanism ready; numbers need domain/operator input) [OQ-10].
- Photoperiod actuation mechanism (driver dim-to-off vs relay switching) → blocks R2
  light.schedule on real hardware [OQ-14].
- LED fixture model/wattage (excluded from rack BOM) → blocks real-energy estimation
  accuracy + R2 commissioning test [OQ-15].
- Food-safety/compliance for selling microgreens in India (FSSAI license, hygiene
  testing, packaging/shelf-life) → blocks R4 inventory-attributes and R5 portal legality
  [OQ-11].

**Engineering-document gaps (hardware program, not platform, but tracked):**
- Chiller capacity (garbled OCR), floor-plan power row 1, earthing conductor size,
  LSH-04 height, rack mass discrepancy, settle/test durations, sensor-count headline →
  physical-fact register + visual pass [OQ-4].
- Terrace slab structural sign-off (explicitly required before tank order); T16 sway
  test; commissioning CV traverse — already documented as open in safety-rules.

**Not blocking, deliberately deferred:** imaging analytics protocols, exotic-plant
definition [OQ-8], saffron program status [OQ-1], B2B commercial mechanics [OQ-13].

## 6. Summary of actions taken by this audit

1. Created `iot-control-systems-engineer` + `ux-product-designer` agents (justified
   separation: control-plane architecture ownership; operator-experience ownership).
2. Deferred 4 agent candidates with explicit triggers (data-architect, R&D-analytics,
   inventory-operations, observability) rather than creating speculatively.
3. Produced the foundation documentation set + ADR-0001…0005 (this docs/ tree).
4. Established the physical-fact discrepancy register (§3.3-8) and OCR-visual-pass
   requirement for a vision-capable session.