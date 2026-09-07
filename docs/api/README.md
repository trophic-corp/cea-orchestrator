# API specifications

**Owner:** `lld-architect` produces API contracts per feature pipeline; `backend-engineer`
implements against them. Populated from the first backend `/ship` (Release 1).

Content to live here when written:

- `operator-api.md` — console backend API (auth, zones, devices, telemetry queries,
  control/command submission, batches, recipes, inventory).
- `device-mqtt-topics.md` — the versioned MQTT topic taxonomy + payload schemas
  (see docs/iot/device-control-model.md; ADR-0004).
- `portal-api.md` — B2B portal API (Release 5), separate identity domain.

Conventions: OpenAPI (or equivalent spec files) once endpoints stabilize; every
safety-relevant endpoint carries a note referencing the security review artifact it
passed. The repository, not Figma or any external tool, is the source of truth for
API contracts.