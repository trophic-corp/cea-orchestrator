# Security considerations — hardware control as a security-sensitive domain

**Status:** proposed · **Date:** 2026-09-07
**Deliverable mapping:** security considerations of the foundation analysis. The pipeline
gate for any implementation is `security-safety-reviewer` (never skipped); this document
defines the architecture it reviews against. Physical safety (interlocks, fail-safes) is
covered in [system-architecture.md](../architecture/system-architecture.md) §"Safety
architecture" — the two are sibling concerns, deliberately kept in separate documents.

## 1. Threat model (why this is not ordinary SaaS)

A compromised dashboard in a normal SaaS leaks data. A compromised CEA control path can
**destroy a crop silently, flood a room, or release CO2 into an occupied space** (the
CO2 asphyxiation context in `safety-rules.json` is explicit: a released cylinder is
lethal well before it is noticed). Principals and threats:

| Threat | Consequence | Primary mitigation tier |
|---|---|---|
| Cloud account/session compromise | Attacker issues destructive setpoints/commands | Cloud cannot originate safety-relevant actions; edge enforces envelope; RBAC; audit |
| Malicious/compromised device on OT network | Forged telemetry, injected commands | Per-device identity + topic ACLs; commands accepted only from the controller; network segmentation |
| Physical/network MITM on telemetry | Wrong decisions from fake data | TLS everywhere; plausibility gates; measured values are advisory to safety loops |
| Insider/operator error | Unsafe setpoints, wrong recipe on a tier | Safety envelope enforcement at edge (bounds from `safety-rules.json`); confirmation UX; audit |
| Firmware supply chain | Malicious image on controllers | Signed images; staged rollout; version pinning; CI provenance |
| Secrets leakage (repo, logs, images) | Full platform compromise | Never in repo (`.gitignore` already excludes `.env`); docker secrets (compose skeleton already models this); scanner gate in CI |
| Backup exposure | Full production data exfiltration | Backups encrypted at rest; access-controlled; export audit-logged |

## 2. Authentication & authorization

**Users (console + portal):**
- Console: email/password + TOTP (from day one — a control system justifies it), session
  expiry, no anonymous anything. WebAuthn when convenient.
- Roles (RBAC, coarse-grained, mapped to personas):

| Role | Read | Operate (control/overrides) | Manage (recipes, batches, inventory) | Admin (users, devices, backup) |
|---|---|---|---|---|
| admin | ✓ | ✓ | ✓ | ✓ |
| grower (P2) | ✓ | ✓ | ✓ | — |
| operator (P1) | ✓ | ✓ (within envelope) | batches/harvest only | — |
| technician (P4) | ✓ | device actions only | — | device fleet + firmware |
| viewer (P3/P5-like) | ✓ | — | — | — |

- B2B portal accounts are a **separate identity domain** (customer accounts, no console
  roles, ever). Cross-domain trust: none.

**Devices:**
- Unique identity created at **provisioning** (commissioning flow, UC-04): device id +
  individual credential (client certificate via mTLS preferred; per-device PSK acceptable
  first-phase compromise documented in an ADR when firmware work starts).
- A device is *untrusted until claimed*: an unclaimed device can advertise capabilities
  but is wired to no zone and receives no commands.
- Credential rotation supported without re-flash (enrollment channel).
- Lost/stolen/replaced device: revoke identity; the registry records the hardware
  fingerprint; a device that reappears with a mismatched fingerprint is quarantined, not
  merged.

## 3. Transport & network architecture

- **MQTT over TLS (8883)** — already the documented intent (`docker-compose.prod.yml`).
  Broker (Mosquitto) enforces per-device topic ACLs: a device may publish only to its own
  telemetry/state topics and subscribe only to its own command/config topics.
- **Network segmentation:** OT segment (controllers, sensors, broker) has **no inbound
  exposure**; the backend reaches devices through the broker; remote access (P4 from
  Coimbatore) comes via the backend over VPN/TLS, never direct-to-device.
- Cloud-to-edge is **outbound-only from the edge's perspective** where possible
  (edge-initiated connections), so the OT network does not need inbound holes.
- **The safety layer is not on any network:** hardwired interlocks (E-stop circuit,
  fail-safe solenoid states, mechanical float valve — `safety-rules.json` /
  `electrical_fail_safes` and `valve_and_actuator_fail_safe_states`) function with zero
  software and zero connectivity. No network-borne compromise can energize a fill
  solenoid open on power loss.

## 4. Command authorization (the core control-path rule)

1. All actuation flows through the **edge controller** (desired-state store + schedules
   + interlocks). The cloud/backend never writes directly to a device; it *proposes*
   configuration; the controller validates and applies.
2. Every command carries: actor identity (user or system component), the change,
   capability target, and a monotonic sequence. The controller **rejects** anything
   outside the **safety envelope** (bounds sourced from `safety-rules.json`: e.g., a
   recipe demanding PPFD beyond the photoinhibition bound, a flood schedule denser than
   the envelope, CO2 dose requests ignoring the 5000 ppm alarm interlock) — regardless
   of who asked, including admin.
3. Manual overrides are time-boxed, audited, and revert automatically (device-control
   model §5) — an attacker (or an operator) cannot silently pin a bad state forever.
4. Invalid commands are rejected with a machine-readable reason and logged; repeated
   rejections from one source raise an alert (automation ≠ retry storm).

## 5. Audit logging

- Append-only audit log, separate storage from operational data; covers: authn events,
  config/recipe changes (with before/after), command issuance (who/what/when + device
  ack result), overrides, alert acks/resolves with reasons, backups/exports, user/role
  changes, device provisioning/revocation.
- Batch-relevant events are *also* mirrored as batch events (dual-write is deliberate:
  audit answers "who did it", batch timeline answers "what happened to the crop").
- Audit records are part of the export bundle (backup requirement).

## 6. Firmware update security

- Images **signed**; controller verifies signature before applying; unsigned builds
  cannot be flashed OTA (dev hardware may allow local flashing with a physical strap —
  a deliberate, visible escape hatch).
- **Staged rollout:** one device/rack first, soak period, then fleet; automatic rollback
  on failed health check post-update (dual-partition A/B).
- **Update policy as a safety rule:** controllers do not apply updates mid-flood-cycle
  or while a safety interlock is active; update windows are configurable; power-loss
  during update must leave the device in a safe (fail-safe states honored) state.
- Supply-chain: build in CI, version + commit hash embedded in the image, provenance
  recorded in the release record.

## 7. Secrets management

- Repo: zero secrets (enforced by review + scanner; `.env` already ignored).
- Deployment: docker secrets files / environment from a restricted source; the compose
  skeleton's password-file pattern is the baseline.
- Edge: device credentials in secure element/encrypted NVS (ESP32-class capability) —
  never hardcoded firmware keys; per-device, so one leaked image ≠ fleet compromise.
- Rotation: database creds, API keys, broker creds all rotatable without code changes.

## 8. Tenant isolation (future productization)

Decision [ADR-0005](../adr/0005-single-tenant-now-tenant-ready-schema.md): single-tenant
deployment now; **tenant-scoping designed in the schema** (org_id on every core entity,
every query org-scoped from day one) so productization later is a routing/auth problem,
not a migration problem. What is *not* built now: tenant signup, cross-tenant admin,
tenant-aware billing, per-tenant data residency.

## 9. Explicit non-goals (this phase)

- No custom cryptography, no homemade protocols.
- No PKI we operate ourselves beyond per-device certificates issued at provisioning
  (offline CA at commissioning time is acceptable at this scale).
- No public attack-surface beyond the console and portal web apps — both behind TLS,
  rate-limited, standard web hardening (CSP, samesite, etc. — release-level detail,
  owned by backend/frontend engineers, verified by `security-safety-reviewer`).