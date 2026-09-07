# Testing strategy

**Owners:** `qa-e2e-validator` (acceptance), `automation-engineer` (harnesses/HIL/sim),
implementation specialists (unit). Populated from Release 1.

Content to live here when written:

- `test-strategy.md` — layers: unit (per service/firmware module), integration
  (compose stack + virtual devices), e2e (PRD acceptance per release), HIL (firmware
  against simulated and real IO — mandatory for kind: firmware/hardware per pipeline).
- `simulation.md` — the virtual facility: device simulators, fault injection matrix
  (offline, stale, implausible, command-fail, reboot), how CI runs it.
- `safety-test-matrix.md` — commissioning + regression tests proving fail-safe states,
  interlock timing (60s leak interlock is an existing QC hold point), E-stop scope.
- `acceptance/` — per-release acceptance criteria execution records.

Rule: every release's acceptance criteria (roadmap) are written *before* build
(working rule 20); `qa-e2e-validator` blocks on them.