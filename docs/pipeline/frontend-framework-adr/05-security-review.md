# 05 — Security & Safety Review: ADR-0009 (operator console frontend framework)

**Reviewer:** security-safety-reviewer
**Date:** 2026-09-08
**Scope reviewed:** `docs/adr/0009-vue3-as-the-operator-console-frontend-framework.md` (status
`proposed`), plus its supporting pipeline artifacts (`01-classification.md`,
`03-domain-brief.md`, `04-hld.md`) in `docs/pipeline/frontend-framework-adr/`.
**What exists in the repo:** nothing beyond the ADR text and its pipeline artifacts.
`frontend/` contains only `.gitkeep` — no `package.json`, no lockfile, no scaffold, no CI
job, no `docker-compose.prod.yml` edit. This review is of a documented technology-choice
decision, not implemented code, and findings are framed accordingly (most are "must be
addressed when scaffold work happens," not "must be fixed in this diff" — there is no diff
to fix).

## 1. Physical safety interlocks (`.github/agentic-rules/safety-rules.json`, ADR-0001)

**PASS — no conflict, and the ADR's own wording actively reinforces the boundary rather
than blurring it.**

- Checked `safety-rules.json` in full: `valve_and_actuator_fail_safe_states`,
  `electrical_fail_safes` (RCBO Type A, ELV boundary, room-level E-stop scope), the 60s
  leak/blocked-drain interlock, VPD/CO2/pH/EC/solution-temperature thresholds, and both
  entries explicitly marked `"TBD — needs domain expert input"` (`dli_mol_m2_day`,
  `ec_operating_target`). A frontend framework/library choice touches none of these — there
  is no valve, actuator, interlock timing, dosing gate, or climate setpoint anywhere in
  ADR-0009's Decision or Consequences text, and no TBD threshold is referenced, filled in,
  or implied. Nothing here triggers the "block until a human supplies the real value" rule.
- **Edge-authority / safety-envelope boundary (ADR-0001) is explicitly and correctly
  preserved in the ADR's own text**, not just assumed by omission. Direct quotes from
  `0009-vue3-as-the-operator-console-frontend-framework.md`:
  - "any client-side validation of capability/safety-envelope bounds ... is a UX convenience
    for fast rejection only — the room controller remains the enforcing authority per
    ADR-0001 point 3 and device-control-model §5, regardless of framework."
  - "This capability is a property of the JSON-Schema-driven form *pattern* generally ...
    not a Vue-specific differentiator."
  This is exactly the language the review brief asked to check for ("does the ADR's wording
  anywhere blur the line" between console-as-informational and console-as-safety-authority)
  — it does not blur it; it repeats the correct allocation twice, once in Context and once
  implicitly in Consequences, and ties it back to the specific ADR-0001 clause and
  device-control-model section rather than asserting it loosely.
- **E-stop scope, valve fail-safe states, RCBO/ELV boundary**: none are touched, referenced,
  or implied to change. `emergency_stop` in `safety-rules.json` remains room-level, hardwired,
  not per-rack, dropping fill solenoids/MV-01/all pumps with drain solenoids de-energizing
  open — nothing in ADR-0009 proposes any software path (console or otherwise) into that
  hardwired layer. Consistent with ADR-0001's "hardwired safety > edge control > facility
  platform > remote/cloud" layering, which the ADR cites directly.
- **Cloud-dependency-for-safety-loop check**: the console is scoped as an HTTP/OpenAPI
  client of the Go backend only, explicitly forbidden from opening its own MQTT connection
  ("the console never opens an MQTT connection of its own, including for 'real-time'
  telemetry"). Since the console was never in the actuation path to begin with (ADR-0001:
  "all actuation decisions execute at the edge"), this framework choice does not create or
  imply a new cloud/internet dependency for anything safety-critical. No finding.

## 2. Supply-chain / dependency risk

**PASS WITH FINDINGS (non-blocking — no lockfile or manifest exists yet to actually audit).**

Checked the named library set (Vue 3, Vite, `@tanstack/vue-query`, openapi-typescript,
openapi-fetch, vue-echarts, uPlot, FormKit, `vue-json-schema-form`, Vue Router, Pinia,
VueUse, `vite-plugin-pwa`) against current (2025–2026) npm supply-chain incident reporting.
None of the specifically named packages appear in the major 2025–2026 campaigns I found
(Shai-Hulud worm, the March 2026 axios maintainer-account compromise, the May 2026
`node-ipc` credential-stealing payload, the June 2026 Mastra/`@mastra` scope takeover, the
`@redhat-cloud-services` and AsyncAPI pipeline compromises). `vite-plugin-pwa` currently
carries a known build-time-only transitive advisory (`serialize-javascript` via
`workbox-build`/`@rollup/plugin-terser`, RCE-class but build-tooling not runtime) that
should be re-checked at scaffold time, not now — versions shift by the time code lands.

None of this blocks recording the framework decision. It does mean the following should be
carried forward as explicit obligations for whoever writes the eventual `frontend/`
scaffold LLD (this is a note for that future stage, consistent with `04-hld.md`'s own
framing that scaffold work is separate and not authorized by this run):

1. **Lockfile discipline is non-negotiable given the current npm threat landscape.** 2025–2026
   has seen repeated worm-style campaigns that self-propagate via compromised maintainer
   accounts and postinstall hooks (Shai-Hulud, node-ipc, axios, Mastra). The eventual
   `frontend/package-lock.json` (or equivalent) must be committed, CI must run `npm ci`
   (not `npm install`), and dependency updates should go through review rather than
   auto-merge, given this codebase's stated agent-authored-code discipline (an agent
   bumping a dependency without a human noticing a hijacked patch release is exactly the
   failure mode these campaigns exploit).
2. **Prefer disabling/auditing install scripts** where the toolchain allows it (npm's
   install-script-blocking mode, or an equivalent audit step in the frontend's independently
   runnable CI job per ADR-0008 rule 4) — several of the incidents above propagate via
   postinstall hooks specifically.
3. **FormKit licensing note**: if the eventual scaffold pulls FormKit *Pro* components
   (rather than the open-source core, or `vue-json-schema-form` instead), FormKit Pro's
   paid tier is typically gated by a project API key referenced in build config. ADR-0009
   does not commit to Pro, so this isn't live yet, but flagging now so nobody later embeds
   that key as a literal string in a committed config file — it belongs in a CI secret /
   environment variable, same discipline as any other credential, not in `frontend/`
   source.
4. **npm-scope naming caution, by analogy with the existing GitHub-org caution in
   `CLAUDE.md`.** If the eventual `frontend/` scaffold introduces an npm workspace/package
   name under a scope (e.g. `@trophic-corp/frontend`), that name is independent of the
   MQTT topic root `trophic/<org>/<site>/…` (ADR-0004) for the same reason the GitHub org
   name is — don't let a future `trophic-corp` → real-org-name rename sweep touch the MQTT
   topic string by accident, and don't assume an npm scope rename is free (registry
   ownership/publish implications) the way a GitHub org rename might be. No package name is
   chosen yet, so nothing to fix now — just don't let this get find-replaced later without
   checking both.

None of the above is a reason to withhold approval of ADR-0009 itself: the ADR is a
framework/library *choice*, not a dependency manifest, and there is no `package.json` or
lockfile in the repo yet for an actual supply-chain audit to bite on.

## 3. PWA / service-worker implications — the one finding worth elevating

**PASS WITH FINDINGS (non-blocking now; should be closed before scaffold work starts, not
after).** This is the substantive finding of this review.

ADR-0009's Decision section commits to `vite-plugin-pwa` for delivery ("PWA via
`vite-plugin-pwa`") and VueUse's online/offline detection "consistent with ADR-0001's
local-first posture," but the ADR is silent on **caching strategy** for anything beyond the
app shell. This matters architecturally, not just at implementation time, for reasons the
ADR's own cited sources already establish:

- `docs/ux/information-architecture.md` §1 (principle, quoted in the doc): the console
  "explicitly states data age... and never renders stale state as current." This is stated
  as an ordered UX principle, not a nice-to-have.
- ADR-0001 Consequences: "console UX must represent edge-authority honestly (stale badges,
  'edge-autonomous' banners)."
- `safety-rules.json`'s `co2_safety_alarm_ppm` (5000 ppm, independent monitor, fail-closed
  solenoid) and the broader alarm/interlock set are exactly the kind of state a service
  worker's default cache-first or stale-while-revalidate strategy (Workbox's common
  defaults, which `vite-plugin-pwa` wraps) could serve from cache as if current, during
  precisely the window — a real connectivity interruption during an active alarm — where an
  operator most needs to know the data in front of them might be stale.

To be precise about severity: the console is explicitly informational-only and the
hardwired/edge layers remain the actual safety enforcement (finding 1, above) — a stale
alarm badge in the console cannot itself cause a physical unsafe state, because the console
was never the enforcement path. This is why the finding is a **should-fix-before-scaffold**
architecture note rather than a block on ADR-0009: it's a UX-honesty gap consistent with
principle 4, not a control-loop gap. But it is exactly the kind of property the review brief
flagged as "architecturally relevant even pre-code" — it's a property of choosing a
Workbox-based PWA approach at all, not an implementation detail to sort out later without
comment.

**What a fix would need to look like** (for the eventual scaffold LLD, not this ADR): the
ADR or its immediate successor decision should state explicitly, the same way it already
did for the MQTT boundary, that:
- API/telemetry/alarm responses are excluded from the service worker's precache and use a
  network-first (or network-only) runtime caching strategy, never cache-first or
  stale-while-revalidate, so a served response is never staler than the last successful
  network round-trip;
- the PWA's offline capability is scoped to app-shell/static-asset availability (so the
  console can render "you are offline, last known state as of Txx:xx" per IA principle 4)
  and explicitly not to serving live device/alarm state as if fresh while offline;
  and
- this is verified in whatever e2e suite eventually covers the console, analogous to the
  "kill the cloud" test ADR-0001 already commits the platform to.

Recommend this be folded into ADR-0009 as an explicit Decision/Consequences bullet (mirroring
how the MQTT boundary was made explicit rather than left implicit), or, if the ADR is
considered closed as scoped, tracked as a named condition on the eventual scaffold work so it
isn't rediscovered from scratch. Either is acceptable; leaving it unstated is the gap.

## 4. Secrets handling

**PASS — no contradiction found.**

- The ADR's delivery model ("static bundle served by the Go backend — no Node runtime on
  the facility host") implies no server-side secret material needs to exist inside the
  bundle at all; a static Vue/Vite bundle is client-side code by construction and cannot
  hold anything that needs to stay confidential (nothing in a browser bundle ever can).
- Auth is confirmed session-based per `docs/ux/information-architecture.md` ("Authenticated
  session; role-scoped" for the operator console), consistent with the ADR's silence on API
  keys/tokens in the client. No wording in ADR-0009 implies embedding a credential,
  long-lived token, or cloud API key in the client bundle.
- openapi-fetch/`@tanstack/vue-query` are described purely as a typed-client/caching layer
  against the backend's own OpenAPI surface — no third-party API key usage is implied.
- The one credential-adjacent risk (FormKit Pro's project key, if that path is ever taken)
  is noted under finding 2 above as a forward-looking caution, not a present violation.

## Verdict

**PASS WITH FINDINGS (non-blocking).**

No hard block. ADR-0009 correctly preserves every physical safety interlock and fail-safe
state in `safety-rules.json` (none are touched), correctly preserves the edge-authority /
console-is-informational-only boundary from ADR-0001 (stated explicitly in the ADR's own
text, not just assumed), does not create a cloud dependency for any safety-critical control
loop, and does not imply embedding secrets in the client bundle. The pipeline may proceed
past this gate for the ADR as scoped (framework/tooling decision, no scaffold).

Two items should be carried forward, neither blocking record-the-decision-now:

1. **Elevate the PWA caching-strategy gap (§3 above) into an explicit ADR or pre-scaffold
   decision** before any actual service-worker configuration is written — this is the one
   finding with real safety-UX weight (principle 4 / stale-state honesty) and should not be
   left to whichever agent happens to configure `vite-plugin-pwa` first without this
   context.
2. **Fold the supply-chain hygiene obligations (§2 above — lockfile/`npm ci`/install-script
   auditing, FormKit Pro key handling if applicable, npm-scope-naming caution) into the
   eventual `frontend/` scaffold LLD** as concrete CI/build requirements, given the elevated
   2025–2026 npm worm/compromise landscape and this codebase's agent-authored-code
   discipline.

Re-review is not required before this ADR is merged/accepted, but **is** required once
actual scaffold/CI work for `frontend/` is proposed (new code, new manifest, new CI job) —
that is a new artifact this review has not seen and cannot pre-clear.
