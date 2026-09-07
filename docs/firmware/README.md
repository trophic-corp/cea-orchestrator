# Firmware documentation

**Owner:** `firmware-engineer` (implementation) with `iot-control-systems-engineer`
(control architecture). Populated from Release 1 firmware work.

Content to live here when written:

- `controller-architecture.md` — rack/room controller responsibilities, fail-safe boot
  sequence, schedule execution, interlock logic, watchdog design.
- `mqtt-contract.md` — device-side view of the ADR-0004 topic taxonomy.
- `ota-policy.md` — signed images, staged rollout, rollback, update-window rules
  (security-architecture.md §6).
- `device-profiles/` — capability profile schema + shipped profiles per device type.

Ground truth for physical behavior: `.github/agentic-rules/safety-rules.json`
(de-energized states are law) and the Rack A engineering docs in `knowledge/cea/cad/`.