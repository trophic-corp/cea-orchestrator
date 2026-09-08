# 0004. MQTT (Mosquitto) + PostgreSQL/TimescaleDB as the single data/transport backbone

**Status:** accepted
**Date:** 2026-09-07 · **Accepted:** 2026-09-08 — owner approval at Phase 1 handover, approved as written
**Pipeline artifact:** docs/product/2026-09-07-workspace-audit.md (foundation analysis)

## Context

The platform needs: device transport (telemetry up, config/commands down) that works
locally and tolerates disconnection; time-series storage for telemetry and desired-state
history; relational storage for entities/batches/inventory/audit. Constraints: single
facility initially (one host, docker-compose), operator with no dedicated devops staff,
local-first reliability (NFR-REL-01/02), budget and complexity discipline (working
rules 8, 10), and a documented prior intent — `docker-compose.prod.yml` already sketches
Mosquitto (MQTT over TLS :8883, "room controllers talk to this locally; cloud sync is a
separate, optional client") and TimescaleDB (pg16), and Phase A §1.7/2.4 prescribes
networked sensors logging to local + cloud DB with per-zone dashboards.

## Decision

0. **Two transport domains, mirroring the documented hardware.** In-room, the field bus
   is Modbus RTU over RS485 multi-drop (rack controllers, I/O nodes → room controller)
   — fixed by the Rack A engineering set, not ours to redesign. The facility backbone
   (room controller ↔ platform, and everything above) is MQTT:
1. **Mosquitto (MQTT 3.1.1/5, TLS :8883)**, broker on the facility host, is the
   facility backbone transport. The **room controller is the edge MQTT client**
   (single authoritative bridge from field bus to platform); backend services and any
   later remote/cloud reader are separate clients with their own ACLs. No service ever
   talks to a rack controller or I/O node directly — the room controller is the only
   path, which keeps edge authority (ADR-0001) enforceable at one choke point.
2. **PostgreSQL 16 + TimescaleDB** in a single database instance serves both relational
   core and time-series (hypertables). One engine, one backup path, one HA story at
   this scale.
3. Topic taxonomy is versioned and part of the contract (device-control-model §4):
   `trophic/<org>/<site>/<rack>/<tier|room>/<device>/telemetry|state|cmd|ack` with
   retained `state/desired` messages so devices and UIs recover last-known desired
   state after reconnects.
4. The edge controller subscribes to its zones' `cmd/config` topics; the **backend
   proposes** configuration via the broker; the controller validates (capability +
   safety envelope) and acknowledges (command pipeline detail in device-control-model
   §5). Backend does not talk to devices any other way.
5. Cloud/remote access, when added, is **an additional MQTT client over a VPN/relay**
   reading/writing through the same broker policies — not a second transport.

## Consequences

- One host runs the whole stack (compose skeleton already shaped for this); backups and
  restore are single-store procedures.
- Disconnection tolerance is native: broker local, devices buffer, backend re-syncs with
  sequence-gap detection.
- TimescaleDB continuous aggregates serve dashboards without an external rollup system.
- Future scale-out (multi-facility, heavy portal) has clean seams: replicate/ship
  hypertables later; broker federation if needed; but none of that is built now.
- MQTT QoS/retention choices become reliability-relevant and must be reviewed by the
  `security-safety-reviewer` for anything safety-adjacent (e.g., command topics must
  never rely on QoS0).

## Alternatives considered

- **HTTP REST + batch upload**: simpler to debug, but polling for commands adds latency
  and complexity, reconnect semantics are worse, and the source research assumes
  pub/sub telemetry; rejected as primary (may exist as read-only external API later).
- **Separate TSDB (InfluxDB/VictoriaMetrics) + PostgreSQL**: splits the transactional
  boundary (batch + telemetry joins), doubles ops surface; TimescaleDB at this scale is
  well within its sweet spot; revisit only with sustained >~10× projected ingest.
- **Kafka/stream platform**: unjustifiable operational cost at one facility; revisit
  at multi-facility (data-architecture §7).
- **Cloud-first broker (devices dial out to cloud MQTT)**: violates local-first
  reliability (NFR-REL-01) and the documented architecture; rejected outright.