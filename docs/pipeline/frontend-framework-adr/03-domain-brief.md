# 03 — Domain brief: ADR-0009 (operator-console frontend stack) consistency check

**Reviewer:** systems-integration-reviewer
**Date:** 2026-09-08
**Mode:** consistency check against accepted/proposed ADRs and `docs/ux/information-architecture.md`
(not a Product & Domain Council reconciliation — see below for why).

## Why there is nothing to reconcile

Step 2 (Product & Domain Council fan-out) was skipped for this run, and correctly so: the
requirement is a frontend framework/build-tooling choice (Vue 3 + Composition API +
TypeScript + Vite, plus the supporting library set), and none of botany, electronics,
mechanical, manufacturing, or product-strategy have domain claims that bear on it. There
are no tolerances, dimensions, material constraints, or crop-science thresholds in play —
this is a pure technical-architecture decision about how the operator console is built, not
what it does. `docs/pipeline/frontend-framework-adr/02-domain-notes/` does not exist (only
`01-classification.md` is present in the artifact directory), which is consistent with the
classifier's own framing of this as a stack decision analogous to ADR-0006, not a
domain-fact-gathering exercise. I checked `.github/agentic-rules/safety-rules.json` directly
regardless of that expectation (per my standing instruction to spot-check rather than trust
absence-of-relevance claims) — it contains no frontend/console/UI/framework entries; every
threshold in it is a physical/process value (VPD, CO2, pH/EC, valve fail-safe states, RCBO,
leak interlock timing), none of which a frontend framework choice touches. So my actual job
here is the consistency check version of this role: does the proposed stack contradict
anything already `accepted` (or `proposed`, in ADR-0006's case)? Findings below, one per ADR
named in the brief, plus the IA document.

## ADR-0001 — Edge-first local control autonomy

**No conflict.** ADR-0001's entire claim is about *where actuation decisions execute*
(edge: rack + room controllers) and what layer is authoritative when connectivity is lost.
A console framework choice is orthogonal to that by construction: Vue vs. React vs. Svelte
all produce a browser client that talks to the backend over HTTP (OpenAPI) and, per
ADR-0001 consequence 5, must "represent edge-authority honestly (stale badges,
'edge-autonomous' banners)" — a UX/behavior requirement, not a framework requirement. Any of
the three candidate frameworks can render staleness indicators and offline-tolerant UI
(ADR-0001 + IA principle 4). The reactivity-model argument in the requirement (Vue's fit to
per-tier telemetry updates) is a *quality-of-implementation* argument for building that
behavior well, not evidence that the other frameworks couldn't. Nothing in ADR-0001 depends
on frontend framework identity.

## ADR-0002 — Capability-based device model

**No conflict — checked against the actual capability schema shape, not asserted.**
ADR-0002's capability classes (`device-control-model.md` §3) carry small, flat-to-shallow
parameter sets: `light.dim {min,max}`, `irrigation.flood {depth_mm,dwell_min,cycle}`,
`valve.control {failsafe_state}` (enum), `co2.enrich {target,alarm}`,
`dosing.fertigation {profile}`. These are exactly the shapes JSON-Schema-driven form
libraries are built for: numeric fields with min/max validation, enums, and shallow nested
objects — none of the classes require deep recursive structures, conditional schema
composition beyond what JSON Schema `oneOf`/`enum` handles, or dynamic layout logic outside
what a schema-driven renderer exposes via slots/custom components. FormKit and
`vue-json-schema-form` both support conditional/enum-driven fields and custom field
components, which covers ADR-0002's actual requirements:

- "a control appears only when the capability exists" (decision point 2) → schema-driven
  forms only render fields present in the schema; a zone missing a capability simply
  doesn't get that field. This is the native behavior of the pattern, not a workaround.
- "recipe parameter is applied or explicitly recorded as 'not applied — lacks capability'"
  → requires the form layer to diff the recipe schema against the zone's capability union
  and render an explicit non-applied state for the difference; this is application logic on
  top of the schema-driven form (a wrapper component), not something FormKit/vue-json-schema-form
  give you for free. Worth flagging to the HLD stage as a concrete build item, not a
  contradiction with the framework choice.
- "profiles are forward-compatible... unknown capability class... recorded and ignored" →
  the form layer must tolerate schema versions/classes it doesn't recognize without
  crashing; this is a rendering-layer requirement (render nothing / a generic fallback for
  unknown `$id`/class), achievable in a schema-driven renderer but again an implementation
  responsibility, not automatic.
- Safety-envelope validation (IA §4 J4: "confirmation dialog showing safety envelope") is a
  UI-level mirror of the edge-authoritative validation in ADR-0001/device-control-model §5
  — the console's schema-driven form can encode `safety-rules.json` bounds as JSON Schema
  `minimum`/`maximum`/`enum` constraints for **UX purposes only** (fast client-side
  rejection), while the edge remains the enforcing authority per ADR-0001 point 3. This
  needs to be stated explicitly in the ADR/HLD so nobody mistakes console-side validation
  for the safety boundary.

Conclusion: the claim "Vue's schema-driven form approach can represent capability-aware,
envelope-validated UI" holds, but it is a claim about the *pattern* (JSON-Schema-driven
rendering), not a Vue-specific property — React has equivalent libraries (rjsf), so this
argument doesn't uniquely favor Vue, only confirms it as sufficient. That's fine for this
consistency check (I'm verifying feasibility, not re-litigating the framework tradeoff) but
worth the HLD noting so the ADR's "architectural fit" rationale doesn't overstate this
particular point as Vue-differentiating.

## ADR-0004 — MQTT/Timescale/Postgres backbone

**No conflict, and the requirement as stated respects the boundary.** ADR-0004 decision
point 1 is explicit: "No service ever talks to a rack controller or I/O node directly — the
room controller is the only path" and backend/cloud readers are separate MQTT clients with
their own ACLs — it does not name the console as an MQTT client at all. The requirement
text ("served as a static bundle by the Go backend") and ADR-0008 rule 1 ("OpenAPI is the
contract of record between `backend/` and `frontend/`") together imply the console talks to
the backend's HTTP/OpenAPI surface only, never the broker directly. I searched the
requirement and classification for any mention of a browser-side MQTT client (e.g.
websocket-bridged MQTT for live telemetry) and found none — `@tanstack/vue-query` +
openapi-fetch is a polling/HTTP pattern, consistent with backend-mediated access. **This
should be stated as an explicit constraint in ADR-0009's Decision section** ("the console
never opens an MQTT connection; live/near-live telemetry is backend-mediated, e.g. polling
or a backend-owned websocket/SSE relay") rather than left implicit, since it's exactly the
kind of boundary a future contributor could accidentally violate by reaching for a
browser MQTT.js client for "real-time" telemetry. Not a contradiction — a gap to close by
being explicit, flagging it for `hld-architect`.

## ADR-0005 — Single-tenant now, tenant-ready schema

**No frontend implication, confirmed.** ADR-0005's decisions are entirely data-layer (org_id
on every table, org-scoped query-layer discipline, no org-switching UI, no tenant billing
UI). The one place it could reach into frontend territory — "do not build... a
fleet-management console" — is a scope statement about *what screens get built*, not about
*what framework builds them*, and this /ship run is explicitly ADR-only (no screens). No
tension with Vue/TS/Vite.

## ADR-0006 — Go as the backend language

**Two claims to verify, both hold:**

1. **OpenAPI-as-contract is satisfiable with Vue+TS.** ADR-0006's "Harder" section commits
   to "shared types must be generated from the OpenAPI spec rather than imported." The
   requirement's stack includes `openapi-typescript` (spec → TS types) +
   `openapi-fetch` (typed fetch client generated from those types) — this is the idiomatic
   Vue/TS-ecosystem realization of exactly that contract, not a workaround bolted onto a
   framework that assumes something else. `@tanstack/vue-query` wraps the generated client
   for caching/invalidation; nothing here requires reaching into `backend/` source or
   sharing types by file path, which would violate ADR-0008 rule 1. Confirmed satisfiable.
2. **"Frontend is TypeScript, so the stack is not single-language" is framework-independent,
   confirmed.** ADR-0006 states this consequence generically, not tied to React. Vue 3 with
   `<script setup>` + TypeScript is exactly as TypeScript as a React+TS console would be —
   the two-language-stack consequence ADR-0006 already accepted is unchanged by which
   TS framework wins. No update to ADR-0006 is implied or needed by ADR-0009.

## ADR-0008 — Monorepo with named split triggers

**No conflict; the requirement text already assumes the disciplines this ADR requires.**

- **Rule 4 (own build/dependency manifest, independently runnable CI):** the requirement
  specifies Vite + its own toolchain under `frontend/`, which is a self-contained
  `package.json`/lockfile by construction — no shared manifest with `backend/` implied.
  Satisfiable as stated. (Not built this round per the owner's ADR-only scoping, but the
  ADR's Decision section should say the eventual scaffold must honor this, mirroring how
  ADR-0006 pre-committed Go's CI/compose obligations before the backend was built.)
- **Rule 3 (no cross-directory imports):** nothing in the requirement's library list
  (vue-query, openapi-fetch, vue-echarts/uPlot, FormKit, Vue Router, Pinia, VueUse,
  vite-plugin-pwa) needs to import from `backend/` or `firmware/` source — all are
  frontend-only libraries or consume the generated OpenAPI client. Satisfiable.
- **Named split triggers T1–T6:** I checked each. T1 (OQ-3 lands on SBC/IPC) — a frontend
  framework choice has no bearing on room-controller compute; doesn't trip it, and isn't
  tripped by it. T2 (aquascaping work begins) — IA principle 6 keeps CEA and aquarium
  navigation/surfaces separate but IA §2 shows both are the same "operator console" /
  future "aquarium suite" *product family* under discussion, not evidence that this ADR
  itself starts aquarium work — it doesn't. T3 (product firmware ships on customer
  hardware), T4 (external partial access), T5 (KiCad/STEP binaries), T6 (firmware OTA
  cadence decouples) — none are about frontend framework selection at all. **This ADR does
  not trip any split trigger.**

## `docs/ux/information-architecture.md` §6 — no premature coupling

**Confirmed clean, and worth stating precisely why.** §6's gate is specific: "Hi-fi Figma
work is explicitly deferred until this IA + journeys J1–J7 are reviewed and Release 1
console scope is frozen." That gate is about *visual/interaction design and screen scope*
(Home/Grow/Batches/Recipes/.../Settings, the J1–J7 flows) — it says nothing about, and does
not gate, the *framework/build-tooling* decision underneath wherever those screens
eventually get built. ADR-0009 as scoped (framework + library choices only, no screens, no
scaffold this round per the owner's explicit ADR-only decision this session) doesn't touch
navigation IA, journey flows, or Figma at all. Table row 1 of §6 ("Functional requirements,
IA, flows, wireframes... Repository... Reviewable, diffable") and row 3 ("Visual design...
Figma, when authorized") are both untouched by this decision. No premature coupling.

One adjacent observation, not a conflict: §6 row 4 references Figma-exported design tokens
as the single path into implementation ("avoids visual drift"). Whatever component-styling
approach eventually pairs with Vue (e.g. a CSS framework or component library) will need to
consume that token export path when it exists — this is a note for whoever scaffolds the
project later, not a gap in ADR-0009's framework-only scope now.

## Overall verdict

**No blocking contradictions found.** ADR-0009, scoped as a framework/tooling-only decision
per the owner's explicit instruction this session, is consistent with ADR-0001, ADR-0002,
ADR-0004, ADR-0005, ADR-0006, ADR-0008, and `docs/ux/information-architecture.md` §6 as
they currently stand (`accepted` except ADR-0006, which is `proposed`).

Two items are not contradictions but should be carried into the ADR-0009 draft explicitly,
so `hld-architect` doesn't leave them implicit where a future contributor could drift:

1. **State the MQTT boundary explicitly** (ADR-0004 cross-check above): the console is an
   HTTP/OpenAPI client of the backend only, never an MQTT client, even for "real-time"
   telemetry — that's a backend-mediated concern (polling via vue-query, or a future
   backend-owned relay), not a console one.
2. **Distinguish "schema-driven form rendering is sufficient" from "Vue uniquely enables
   it"** (ADR-0002 cross-check above): the capability-model fit is real and verified against
   the actual capability parameter shapes, but it's a property of the JSON-Schema-driven
   form *pattern* generally (available in the React ecosystem too), not a Vue-specific
   differentiator — the ADR's alternatives-considered section should rest its case against
   React on the architectural-fit/reactivity/authorship grounds already named in the
   requirement, not lean on capability-model support as a tiebreaker it doesn't uniquely
   provide.

Neither item blocks ADR-0009 from being drafted; both are refinements for `hld-architect`
to fold into the Decision/Consequences text.
