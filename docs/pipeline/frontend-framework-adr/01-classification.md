# 01 — Classification: Operator console frontend framework/stack

**Reviewer:** architecture-reviewer
**Invoked by:** `/ship`
**Date:** 2026-09-08

## kind: `frontend`

Confirmed against repo state, not just the requirement text. `frontend/` contains only
`.gitkeep` — no scaffold exists to conflict with. The requirement is a stack decision
(Vue 3 + TS + Vite + supporting libraries) plus, per the roadmap, an eventual operator
console under that stack (`docs/roadmap/release-plan.md` Phase 1 deliverable #2). No
backend, firmware, or hardware source is touched by the decision itself: the console
consumes the backend only through the generated OpenAPI client (ADR-0008 boundary #1),
so this stays cleanly inside `frontend/`. `kind: frontend` is correct as proposed.

## change_class: `greenfield` — confirms `/ship` is the correct entry point

Checked `docs/adr/` (0001–0006, 0008; 0000 is the template) for any prior coverage of
frontend framework/stack. None exists:

- ADR-0006 (Go as backend language) is the nearest analog in spirit — a language/framework
  choice with alternatives-considered — but it scopes `backend/` only and says nothing
  about `frontend/`.
- ADR-0008 (monorepo, named split triggers) governs *boundary discipline* between
  `frontend/`, `backend/`, and `firmware/` (OpenAPI as contract, no cross-directory
  imports, independent build manifests per subsystem) but is silent on *which* frontend
  framework lives inside that boundary.
- 0007 is reserved in `docs/decisions/open-questions-risks-next-actions.md` for the
  still-open OQ-3 room-controller-platform decision (closes as ADR-0007) — unrelated
  subsystem (room controller compute platform), not this.

No ADR touches frontend framework territory at all → this is genuinely `greenfield`.
`/ship` (full Product & Domain Council → HLD/ADR → security review → LLD → build → ship)
is the correct pipeline, not `/extend`. **Next available ADR number is 0009** (0007 is
reserved for OQ-3/ADR-0007, 0008 is taken by the monorepo decision) — flagging this now so
`hld-architect` doesn't collide with 0007 out of an assumption that it's free.

## Safety-relevance: **No**

Checked `.github/agentic-rules/safety-rules.json` for fail-safe states, dosing/CO2/
electrical thresholds, and valve/interlock entries (CO2 alarm/fail-closed solenoid, EC
gate G1, leak interlock solenoid close time, MV-01/FV-01/trough-dump/make-up-float
fail-safe states). None of these are touched by a frontend *framework* choice — this
requirement doesn't specify UI behavior for any control surface, just the tooling the
console is built with. Independent of this flag, `security-safety-reviewer` is mandatory
for `/ship` regardless (per the orchestration model), so this doesn't change the
downstream pipeline shape — noting it only for completeness per my own instructions.

## Flag 1 for the pipeline coordinator: does the OQ-3 Phase 1 build pause apply here?

`docs/roadmap/release-plan.md` records that the owner has **paused the Phase 1 build**
pending OQ-3 (room controller compute platform, closing as ADR-0007) — see the roadmap's
Phase 1 scope-lock note and action-item #9 ("Build is PAUSED at owner's direction pending
OQ-3 ... Resumes with the Phase 1 `/ship` once ADR-0007 lands"). This needs unpacking
rather than blanket-applying, because two different things are being asked for here:

- **Drafting the ADR itself** (recording the Vue 3/TS/Vite decision with rationale and
  alternatives) is analogous to what already happened with ADR-0006: that ADR was drafted
  and recorded (status `proposed`) *despite* the pause, on the explicit reasoning that
  "OQ-3 cannot reverse this... backend work in Go is not gated on it." The same reasoning
  applies here, arguably more cleanly: OQ-3's live options (Linux SBC, industrial/panel
  PC, or collapsing onto the facility host) are all about what the *room controller*
  runs on. The operator console is explicitly served from the facility host (static
  bundle behind the Go backend) — it has no dependency on room-controller compute at all.
  **I don't see a path by which any OQ-3 outcome reverses a frontend framework choice.**
  Recommendation: the ADR can be drafted now, held at `proposed` per the same convention
  ADR-0006 uses, without waiting on ADR-0007.
- **Actual scaffold/build work** (a real Vite project under `frontend/`, wiring into
  `docker-compose.prod.yml`, a CI job) is a different matter. Roadmap deliverable #2
  ("Operator console v0") is one of the five concrete Phase 1 deliverables that action
  item #9 describes as paused as a whole, and deliverable #1's compose/CI wiring is
  shared infrastructure the backend spike already touched. I don't think it's my call
  (the classifier's) to decide whether "ADR + inert scaffold with zero backend/firmware
  coupling" is close enough to pure paperwork to proceed anyway, or whether it counts as
  build work that should wait for ADR-0007 like everything else in Phase 1. **Flagging
  this explicitly rather than deciding it**: the pipeline coordinator should choose
  between (a) ADR only, scaffold deferred until ADR-0007 lands, or (b) ADR + scaffold now
  on the grounds that frontend tooling setup has no cross-subsystem coupling per
  ADR-0008's boundary rules and can't be "built on a wrong room-controller assumption."

## Flag 2 for the pipeline coordinator: ADR-only vs. ADR+scaffold vs. full console screens

Independent of the OQ-3 question above, there is a second, separate scoping question
about *how much* to build once/if building starts:

`docs/ux/information-architecture.md` §6 states hi-fi work is **explicitly deferred**
until the IA and journeys J1–J7 are reviewed and Release 1 console scope is frozen (the
roadmap gate) — and I see no evidence in this repo that review/freeze has happened yet.
That means building the actual Phase 1 screens (Home, Grow, Devices, Alerts, Settings —
roadmap deliverable #2) now would be building ahead of that IA review gate.

The requirement as given to me is explicitly framed as a **framework/stack decision**
(mirroring ADR-0006), not a request to build the five console screens. I recommend the
downstream HLD stage scope this as **ADR + project skeleton only** (Vite+Vue3+TS config,
lint/build tooling, dependency manifest, CI job wired independently per ADR-0008 rule 4) —
explicitly *not* the Home/Grow/Devices/Alerts/Settings screens — until the IA review gate
above is satisfied. This is a recommendation for the next stage to confirm or override,
not a decision I'm making here.

## Rationale (summary)

This is a straightforward greenfield framework/stack decision with no existing ADR
coverage, correctly routed through `/ship`. It is not safety-relevant. The two open
questions that materially affect scope — whether the Phase 1 build pause reaches ADR
drafting and/or inert scaffold work, and how far to build (ADR-only vs. ADR+scaffold vs.
full console) — are flagged above for the pipeline coordinator and downstream HLD stage
respectively, rather than resolved unilaterally here, per this agent's mandate to classify
rather than design.
