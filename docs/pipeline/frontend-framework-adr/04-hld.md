# 04 — HLD: Operator console frontend framework (ADR-0009)

**Author:** hld-architect
**Date:** 2026-09-08
**Pipeline artifact:** docs/pipeline/frontend-framework-adr/
**Produces:** docs/adr/0009-vue3-as-the-operator-console-frontend-framework.md

## Why this HLD is short

Normal HLD scope is subsystem boundaries, interfaces, control/data flow, and rejected
tradeoffs for a design. This run has no subsystem to design: `frontend/` contains only
`.gitkeep`, and per the owner's explicit decision recorded upstream (01-classification.md,
03-domain-brief.md), **this `/ship` run's deliverable is the ADR only** — no scaffold, no
`docker-compose.prod.yml` edit, no CI job. There is no data flow to diagram because no code
exists to flow through yet. What follows is the minimum HLD content that actually applies:
what is being decided, why it needs ADR treatment now despite zero code, and an explicit
scope boundary so no downstream agent reads more into this run than was authorized.

## What is being decided

A **framework/tooling choice** for the operator console client, not an implementation:

- Vue 3 (Composition API, `<script setup>`) + TypeScript + Vite as the framework/build core.
- A named supporting-library set (`@tanstack/vue-query`, openapi-typescript + openapi-fetch,
  vue-echarts + uPlot, FormKit or vue-json-schema-form, Vue Router, Pinia if needed, VueUse,
  vite-plugin-pwa) consistent with that core.
- Deployment shape: a static bundle served by the Go backend — no Node runtime on the
  facility host, no SSR.
- An explicit boundary restatement (folded in from the domain-brief consistency check): the
  console is an HTTP/OpenAPI client of the backend only; it never opens an MQTT connection
  itself. This was implicit in the requirement's library list (vue-query + openapi-fetch is
  a polling/HTTP pattern) and is now stated directly in the ADR so a future contributor
  doesn't reach for a browser MQTT.js client for "real-time" telemetry and inadvertently
  cross the ADR-0004 boundary (only the room controller and backend/cloud readers are MQTT
  clients; the console is not named as one).

What is **not** being decided here: any of the five Phase 1 console screens (Home, Grow,
Devices, Alerts, Settings), any visual/interaction design (gated separately by
`docs/ux/information-architecture.md` §6 until IA/journey review and Release 1 scope
freeze), or the B2B portal's (R5) or room-controller local UI's (OQ-3/future ADR-0007)
framework — both are separate applications this ADR does not bind (see Consequences below).

## Why this needs ADR treatment now, despite no code existing

The precedent is ADR-0006, drafted and recorded at `proposed` before a single line of
backend Go existed, on the reasoning that the decision itself was settled and future work
would need to build on it rather than re-derive it. The same reasoning applies here, and
arguably more cleanly:

- **It gates all future frontend work.** Every subsequent console PR (scaffold, screens,
  component library choice) either conforms to this decision or re-opens it explicitly.
  Recording it now means the eventual scaffold (Release 1 deliverable #2, currently paused
  behind OQ-3 per the roadmap) starts from a settled framework rather than an implicit
  inheritance the way the Node.js backend assumption was inherited from the bootstrap
  scaffold and never weighed (see ADR-0006's Context, and the same failure mode ADR-0008
  names for the monorepo assumption). This ADR exists specifically so "which frontend
  framework" doesn't quietly become "whatever the scaffold generator defaulted to."
- **No OQ-3 outcome touches it.** OQ-3 decides room-controller compute (Linux SBC,
  industrial/panel PC, or facility-host collapse). The operator console is explicitly
  served from the facility host as a static bundle behind the Go backend — it has no
  dependency on room-controller compute. There is no path by which any OQ-3 outcome would
  reverse a console framework choice, so waiting for ADR-0007 would stall a decision that
  cannot be informed by it, the same argument ADR-0006 already made for Go.
- **It is consistent with, not gated by, ADR-0008's monorepo discipline.** `frontend/`
  keeping its own build/dependency manifest (ADR-0008 rule 4) and consuming the backend only
  through the generated OpenAPI client (rule 1) are properties of the *chosen* stack
  (Vite's own toolchain, openapi-typescript/openapi-fetch), confirmed satisfiable by the
  domain-brief review — recording the choice now costs nothing against that discipline.
- **It is consistent with the IA gate, not in tension with it.** `docs/ux/
  information-architecture.md` §6 defers hi-fi visual/interaction design until IA/journey
  review and Release 1 scope freeze. That gate is about screens and Figma work, not about
  the framework underneath wherever those screens eventually get built. A framework decision
  made now does not anticipate or prejudge that review.

## Explicit confirmation: no scaffold or build steps follow from this run

Per the owner's explicit instruction this session, this `/ship` run produces the ADR only:

- No `frontend/` scaffold (no `package.json`, no Vite project, no lint config).
- No edit to `docker-compose.prod.yml`.
- No CI job for `frontend/`.
- No console screens, no component library selection beyond what's named in the ADR's
  Decision section, no Figma/design-token work.

The next stage in this pipeline is `security-safety-reviewer` (mandatory for every `/ship`
run regardless of safety-relevance flag, per the orchestration model), not `lld-architect`
or `frontend-engineer` — there is no low-level design or build to do yet, only a decision to
review for consistency with prior safety-relevant boundaries (it will find the same
MQTT-boundary point the domain brief already surfaced, now made explicit in the ADR).
Scaffold work is Release 1 deliverable #2 and remains paused pending ADR-0007 (OQ-3) per
`docs/roadmap/release-plan.md`, unaffected by this run.

## ADR disposition

**A new ADR is opened: `docs/adr/0009-vue3-as-the-operator-console-frontend-framework.md`,
status `proposed`.** This is not "purely an application of already-decided architecture" —
no prior ADR names a frontend framework, and this choice will outlive the single requirement
it's attached to (every future console PR either follows it or reopens it). It follows the
same "why proposed and not accepted" convention ADR-0006 uses: the decision itself is
settled; it stays at `proposed` until its own PR merges, at which point `docs-writer` (never
this agent) flips it to `accepted`.

## Tradeoffs folded in from the domain-brief review

Two refinements were required by `03-domain-brief.md` and are reflected in the ADR text
rather than left implicit:

1. The MQTT boundary (above) is now an explicit Decision/Consequences statement, not an
   inference from the library list.
2. Capability-aware/schema-driven form support is framed in the ADR's Alternatives section
   as a property satisfied by the chosen stack (and equally available to React via
   react-jsonschema-form/uniforms), not as a Vue-vs-React differentiator. The ADR's actual
   case against React rests on architectural fit to an admin/dashboard genre, the
   proxy-based reactivity model's fit to independent per-tier telemetry updates, SFC
   review-cost/authorship-quality arguments, and Composition API's long-term stability
   record — not on form-library capability, which both ecosystems provide.
