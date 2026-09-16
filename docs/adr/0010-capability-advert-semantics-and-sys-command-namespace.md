# 0010. Capability advert = hardware inventory plus firmware-accepted commands; `sys.*` controller command namespace

**Status:** proposed
**Date:** 2026-09-10 · **Amended:** 2026-09-10 (HLD gate, security review F-1)
**Pipeline artifact:** docs/pipeline/phase-1-foundation/ (`03-domain-brief.md` §3, §6.1; `04-hld.md` §3.3, §4.4; `05-security-review.md` F-1)
**Refines:** [ADR-0002](0002-capability-based-device-model.md) (does not supersede it)

## Context

ADR-0002 says every device advertises "capability classes it can perform" and that "a
control appears only when the capability exists". Phase 1 is the first release that has to
make a device *say* something, and it must do so under a scope lock that forbids all
actuation: the ESP32 skeleton, the virtual devices, the registry, the console and the edge
command router all need one answer to the question **what does "advertise" mean when the
hardware has an actuator the firmware will not drive?**

The Rack A hardware genuinely has, per rack, 4 fill solenoids (NC), 4 drain solenoids (NO),
4 EC fans with tacho, and 4 LED dimming pairs (trophic-contracts v0.2.0
`sensors/instrument-tags.md`). Three Phase 1 needs require that inventory to be in the
advert even though Phase 1 firmware accepts no actuating command:

- The registry must be truthful about the rack so R2 needs no new profiles
  (product-strategy note 3.4-f).
- The commissioning wizard's step 2 checks advertised actuator counts against the
  instrument-tags counts and cross-checks every `valve.control.failsafe_state` against
  `safety-rules.json valve_and_actuator_fail_safe_states`, **blocking activation on a
  mismatch** — a technician needs that information before a rack is trusted (UX note §5).
- The hardware truth-owner's model of the profile already separates "what is fitted" from
  "what this firmware version will accept" (electronics note 1.3.1).

The opposing position (IoT note §0.4) — advertise only sensing + `sys.identify` until
firmware can honour actuator classes, so that the REJECT path is a real test — protects
something important: ADR-0002's "capability absent/incompatible is a first-class outcome"
must be exercised, not stubbed.

Separately, the commissioning "guided test" must be a command that round-trips with an ack
but touches no valve, fan, LED or pump (classification; release-plan lock). The
device-control-model §3 vocabulary has no such class, and two names were in play
(`control.identify` from product/UX, `sys.identify` from electronics/IoT).

The HLD-gate security review (F-1, High) found the first draft of this ADR internally
inconsistent: it made `commands ⊆ capabilities` a validation error while every Phase 1
profile listed `sys.identify`/`sys.ping` in `commands[]` with no `sys.*` entry in
`capabilities[]`, and it exempted `sys.*` from interlock/envelope checks on a property
(non-actuation) that nothing enforced at admission time. The amendment below resolves both
by making `sys.*` members **actions of the already-existing `control.compute` capability**
and by closing the namespace.

## Decision

1. **A capability profile carries two lists, and both are normative:**
   - `capabilities[]` — the **hardware inventory**: every capability class the device is
     physically fitted with, each instance with its `scope`, `tag`, parameters, and
     `hardware: present | optional`. This is ADR-0002's "classes it can perform" read as
     *what is there*. Nothing listed in trophic-contracts `capabilities/absent-by-design.md`
     may appear. Every controller-class device (rack, room, io-node) advertises
     `control.compute {role}` here — it is the device-control-model §3 class for "the
     controller itself".
   - `commands[]` — the **classes and actions this firmware version accepts**
     (`{class, actions[]}`). A capability may be in `capabilities[]` and absent from
     `commands[]`; **the reverse is a profile validation error, with no exemptions.**
2. **Rejection semantics at the edge** (device-control-model §5, `reason_code` v1):
   - class not in `capabilities[]` → `REJECTED{capability_not_advertised}`
   - class in `capabilities[]` but class/action not in `commands[]` →
     `REJECTED{action_not_supported}`
   - a `sys.*` action not in the closed list of point 4 → `REJECTED{action_not_supported}`
   - params outside the advertised parameters → `REJECTED{params_out_of_range}`
   The edge validates against the device's **own** advert, not the registry's copy. The
   backend repeats the check as a fast path (ADR-0001 point 3) and additionally enforces a
   compile-time allow-list of `sys.*` actions it will ever publish.
3. **A control surface appears only when the class is in `capabilities[]` AND in
   `commands[]` AND the device is `ACTIVE`.** This refines ADR-0002 point 2 / FR-CTL-06. The
   inventory is rendered as information (device detail, commissioning verification), never
   as a control.
4. **`sys.*` is a closed namespace of actions of the `control.compute` capability.** It is
   not a capability class of its own. A `sys.*` action is carried in the profile as
   `commands: [{class: "control.compute", actions: ["sys.identify", "sys.ping"]}]` and in a
   command message as `capability: "control.compute", action: "sys.identify"`. The subset
   invariant of point 1 therefore holds for every profile without special-casing. The set is
   **enumerated in `docs/iot/schema/v1/`** (a closed enum, like `reason_code`), and the edge
   rejects any action outside it. Members as of this ADR:
   - `sys.identify {max_duration_s}` — a no-op with an observable, side-effect-free
     indication (controller status LED / virtual "identify active" flag) bounded by the
     advertised `max_duration_s` and enforced at the device; it must never touch a relay
     channel, a 0–10 V pair, a fan PWM line, or a grow LED.
   - `sys.ping` — no I/O; acks with `seq`, uptime, firmware version, clock status.
   - reserved: `sys.backfill` (R2, pull backfill from the room controller's local store;
     reserved name only, not admitted until its ADR amendment).

   **Admission rule.** A `sys.*` action is admitted to the namespace only if it drives no
   output — no relay, PWM, 0–10 V, LED, pump, valve, doser or HVAC signal — and writes no
   stored schedule or last-known-good setpoint. That property is what justifies the
   exemption in the next sentence; a command that touches any output is a crop-control
   class, never `sys.*`. **Adding a `sys.*` member requires amending this ADR** and the
   schema enum in the same PR.
   `sys.*` actions are not subject to interlock or safety-envelope checks (by construction
   they actuate nothing) but are subject to `device_unclaimed`, `expired`,
   `stale_sequence`, `busy`, `params_out_of_range`. Diagnostic metrics such as
   `sys.heap_free` are optional advertised metrics on `control.compute`, not commands.
5. **The Phase 1 commissioning guided test is `sys.identify`.** Every Phase 1 profile —
   virtual rack, virtual room controller, virtual I/O nodes, and the real ESP32 skeleton —
   advertises `control.compute` in `capabilities[]` and
   `commands = [{class: control.compute, actions: [sys.identify, sys.ping]}]` and nothing
   else.
6. Naming: `sys.identify` over `control.identify`. Both need the same §3 registration; the
   `sys.*` prefix keeps controller housekeeping apart from crop-control classes and is the
   name the hardware-side specialists used.

## Consequences

**Easier**
- The registry is truthful about hardware from the first advert; R2 adds actuator classes to
  `commands[]` in a firmware release and changes no profile schema, no registry code, no
  console rendering rule.
- The ADR-0002 "absent/incompatible is first-class" property gets **two** CI tests instead of
  one: `capability_not_advertised` (a class no rack has, e.g. `co2.enrich`) and
  `action_not_supported` (a class the rack has but this firmware does not accept, e.g.
  `light.dim`). The second is the more valuable, because it is the case a real R2 rollout
  will hit mid-fleet.
- Commissioning can cross-check fail-safe states against `safety-rules.json` before a rack is
  activated — information the technician needs regardless of whether software will ever
  drive the valve.
- "No actuation in Phase 1" is enforced where it belongs — at the edge (`commands[]`), at the
  backend (allow-list), and in the console (rule 3) — not by pruning the truth out of the
  advert.
- One validation rule, no exemption path: `commands ⊆ capabilities` is checked identically
  for `sys.*` and for crop-control classes because `sys.*` lives under `control.compute`.
  A device that does not advertise `control.compute` (a hypothetical dumb sensor node)
  correctly cannot be identified or pinged.

**Harder**
- Every consumer of a profile must read both lists; a consumer that renders a control from
  `capabilities[]` alone is a bug, and the console's rendering rule (3) must be tested as
  such.
- Profile validation gains one cross-list invariant (`commands ⊆ capabilities`) and one
  enum check (`sys.*` actions ∈ the closed list).
- Firmware must build `commands[]` from what it actually implements, not from a board table —
  the discovery principle now applies to commands as well as sensing.
- Growing `sys.*` costs an ADR amendment. That is the point: the interlock/envelope
  exemption is only safe while membership is reviewed.

**Must now be respected rather than re-derived**
- device-control-model §3 gains the `sys.*` namespace (as `control.compute` actions) and the
  two-list profile shape (`docs-writer` amends at ship time); `docs/iot/schema/v1/` encodes
  the profile shape and the closed `sys.*` enum.
- The backend's command allow-list and the edge's `sys.*` check both read the same schema
  enum; a `sys.*` action absent from the enum is refused at both layers.
- R2's first actuating firmware version adds classes to `commands[]` only after the relay
  polarity/boot-state bench check (brief §5.4) has passed for that board revision.
- The aquarium product line (ADR-0002 point 5) inherits the same two-list semantics; a tank
  light with `light.dim` in inventory but not in `commands[]` renders no dimmer.

## Alternatives considered

- **Advertise only what the firmware can currently do (IoT §0.4).** Cleanest REJECT test on
  its face, but the registry lies about the rack until R2, the commissioning fail-safe
  cross-check becomes impossible, per-tier instances that the UI needs for Grow/device detail
  disappear, and the more valuable `action_not_supported` case is lost. Rejected.
- **Advertise the full inventory with no `commands[]` and rely on the backend having no
  `cmd` path (product 3.4-f as originally stated).** Puts the "no actuation" guarantee solely
  in the backend, which ADR-0001 says is the *advisory* layer; the edge would ack SUCCESS to
  a bypassed `light.dim` if it ever gained the code. Rejected — the edge must be able to say
  "not in this firmware" itself.
- **Two separate documents — a hardware profile and a firmware profile.** Two artefacts to
  keep in sync per device revision, two schemas to version, and a registry join to answer
  "can this device do X right now". Rejected; one advert with two lists carries both truths.
- **A per-capability `enabled`/`implemented` boolean instead of a `commands[]` list.** Cannot
  express "class advertised, some actions accepted" (e.g. `light.state_report` read-only vs
  `light.dim set`), and does not name actions, which the `cmd` message already carries.
  Rejected.
- **Exempt the `sys.*` namespace from the subset invariant and validate it against a closed
  list (security review F-1 option b).** Closes the inconsistency, but creates a second
  validation path — one rule for crop-control classes, another for `sys.*` — that every
  consumer (edge, backend, registry, console) must implement twice. Rejected in favour of
  hanging `sys.*` off `control.compute`, which every controller already advertises; the
  closed list is kept for the admission rule, not for the invariant.
- **`sys` as its own capability class that every controller advertises implicitly.** Adds a
  class whose only content is "I am a controller", which `control.compute {role}` already
  says; and "implicitly advertised" is exactly the kind of hidden default the discovery
  principle forbids. Rejected.
- **`control.identify` naming.** Cosmetic; rejected in favour of a namespace that keeps
  housekeeping separate from crop-control classes and matches the hardware-side vocabulary.
