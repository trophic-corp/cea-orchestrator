# 06 — Low-level design: phase-1-foundation (part 1 of 2)

**Author:** lld-architect · **Date:** 2026-09-10 · **Pipeline:** `/ship` · **Branch:** `phase-1-foundation`
**Inputs:** `04-hld.md` (post-review revision; §3.3, §6, §7.3, §10.3 current), ADR-0010 and
ADR-0011 (proposed, binding here), `05-security-review.md` F-1..F-15, `03-domain-brief.md`
(§4 defaults, §5 safety, §10.1 carry-forwards), `02-domain-notes/{ux-product-design,
iot-control-systems,botany-horticulture,electronics-hardware}.md`, ADR-0001..0009,
`docs/iot/device-control-model.md`, `docs/security/security-architecture.md`,
`.github/agentic-rules/safety-rules.json` (2026-09-05), trophic-contracts v0.2.0
(`conventions/units.md`, `sensors/instrument-tags.md`, `safety/interlocks.md`,
`safety/valve-fail-states.md`, `capabilities/absent-by-design.md`).
**This document is in two files.** `06-lld.md` (this file): §1 repo layout, §2 wire schema
v1 and `rack-link/0`, §3 database, §4 OpenAPI v1, §5 backend Go interfaces, §6 virtual room
controller. `06-lld-part-2.md`: §7 ESP32 skeleton, §8 frontend, §9 compose and CI, §10 test
plan, §11 defaults and unknowns tables, §12 security-review disposition, §13 flags. Both are
normative; a specialist reads both.

**Scope guard (unchanged from the HLD).** Nothing here publishes a `cmd` outside
`control.compute/sys.identify`, stores a desired state, executes a schedule, or initialises
an actuator output. Every threshold in §3.7 is copied from `safety-rules.json` with key path,
source string and comparator; nothing is invented, and every `[unknown]` stays null.

---

## 1. Repository layout (file/package level)

### 1.1 `backend/` — Go module `github.com/trophic-corp/cea-orchestrator/backend`

The module path tracks the GitHub org placeholder; when the org is renamed the `go.mod`
`module` line and import paths are edited **as one scoped change** (never a global rename,
CLAUDE.md). Go **1.23.x pinned** in `go.mod` `toolchain` and in CI (`go-version-file`).

```
backend/
  go.mod  go.sum                       toolchain go1.23.x; -mod=readonly in CI
  Dockerfile                           multi-stage: stage1 node:20 builds ../frontend (repo-root
                                       context); stage2 golang:1.23 builds ./backend; final
                                       gcr.io/distroless/static; console at /srv/console
  cmd/trophic/main.go                  single binary; subcommands below (spf13/cobra NOT used — stdlib flag + subcommand switch)
  cmd/trophic/serve.go                 serve: api + ingest + alerts + backup scheduler in one process
  cmd/trophic/migrate.go               migrate [--to N] (goose, forward-only)
  cmd/trophic/seed.go                  seed --file <seed.yaml> [--dry-run]
  cmd/trophic/export.go                export --out <dir> [--raw-window-days 30]
  cmd/trophic/restore.go               restore --bundle <file> (--validate | --policy import-as-new|replace)
  cmd/trophic/acl.go                   acl materialise [--out /mosquitto/acl/aclfile]
  cmd/trophic/pki.go                   pki fingerprint <cert.pem>  (registry helper; never signs)
  internal/config/                     env + secrets-file loading (config.go, secrets.go)
  internal/store/                      pgx pool, tx helpers, scope.Org, repository interfaces
    store.go  scope.go  tx.go
    repo_facility.go repo_users.go repo_sessions.go repo_devices.go repo_profiles.go
    repo_telemetry.go repo_lks.go repo_gaps.go repo_events.go repo_alerts.go repo_rules.go
    repo_commands.go repo_audit.go repo_backup.go repo_platform.go
    repotest/        two-org fixture, scoping-coverage gate (§5.9)
  internal/authz/                      password.go (argon2id), totp.go, session.go, rbac.go,
                                       middleware.go, ratelimit.go, audit.go
  internal/registry/                   lifecycle.go (state machine), health.go (two axes),
                                       profile.go (validate + capability index), commissioning.go,
                                       acl.go (materialiser), fingerprint.go
  internal/ingest/                     mqtt.go (paho client, single publisher), topic.go,
                                       envelope.go, crosscheck.go, dedup.go, gaps.go,
                                       plausibility.go, classfallback.go, vpd.go, dli.go,
                                       staleness.go, pipeline.go, outage.go, publisher.go
  internal/alerts/                     rules.go, engine.go (state machine), safety.go (mirror
                                       semantics), notify.go (console + smtp), noisy.go, seed.go
  internal/backup/                     bundle.go, manifest.go, crypt.go (age), export.go,
                                       restore.go, schedule.go
  internal/api/                        server.go (net/http ServeMux, Go 1.22 patterns),
                                       gen/ (oapi-codegen output, committed), handlers_*.go,
                                       envelope.go (value envelope builder), problem.go,
                                       page.go (cursor pagination), static.go (console),
                                       spectest/ (route-vs-spec + request/response validation)
  internal/schema/                     v1 payload structs + JSON-Schema validation (santhosh-tekuri/jsonschema/v6);
                                       loads docs/iot/schema/v1 at test time via TROPHIC_SCHEMA_DIR
  internal/sysaction/allowlist.go      const AllowList = [...]string{"sys.identify"}  (§5.6)
  migrations/                          goose SQL, 0001_… forward-only (§3.1)
  seeds/facility-ooty-r1.v1.yaml       versioned structure seed (D-3)
  testdata/                            goldens copied by CI from docs/iot/schema/v1/golden
```

Dependencies (all pinned in `go.sum`, `govulncheck` in CI): `jackc/pgx/v5`,
`pressly/goose/v3`, `eclipse/paho.mqtt.golang`, `santhosh-tekuri/jsonschema/v6`,
`getkin/kin-openapi` (test-only validation), `oapi-codegen/oapi-codegen/v2` (generation),
`pquerna/otp`, `golang.org/x/crypto/argon2`, `filippo.io/age`, `oklog/ulid/v2`,
`wneessen/go-mail` (SMTP), `robfig/cron/v3` (backup schedule). No ORM.

### 1.2 `frontend/` — Vue 3 + TS + Vite (ADR-0009)

```
frontend/
  package.json  package-lock.json  .npmrc (ignore-scripts=true, save-exact=true)
  vite.config.ts  tsconfig.json  eslint.config.js (no-restricted-imports: ../backend ../sim ../firmware)
  index.html
  src/
    main.ts  App.vue  router.ts  (route table §8.1)
    api/generated/schema.d.ts        openapi-typescript output — COMMITTED, drift-checked
    api/client.ts                    openapi-fetch client, baseUrl /api/v1, credentials: include
    api/queries/*.ts                 vue-query hooks per endpoint group
    shell/AppShell.vue SafetyBanner.vue ConnectionStrip.vue SimChip.vue NavTabs.vue
    components/ValueEnvelope.vue     THE number component (§8.3)
    components/ProvenanceTag.vue AgeText.vue HealthBadge.vue AlertRow.vue CapabilityList.vue
    views/Login.vue Home.vue
    views/grow/RoomView.vue RackView.vue TierView.vue TelemetryChart.vue
    views/devices/FleetList.vue DeviceDetail.vue commission/Wizard.vue Step0..Step5.vue
    views/alerts/AlertCenter.vue ActiveTab.vue HistoryTab.vue RulesTab.vue AlertDetail.vue AckForm.vue
    views/settings/Settings.vue Users.vue Backup.vue System.vue Notifications.vue
                   SafetyEnvelope.vue Facility.vue
    stores/connection.ts             Pinia: last as_of, backend reachable, backoff
    composables/useAsOf.ts usePolling.ts useRole.ts
    pwa/sw-config.ts                 app-shell-only precache (§8.6)
  tests/unit (vitest)  tests/e2e (playwright; run from e2e-sim job)
```

### 1.3 `firmware/`

```
firmware/
  rack-controller/                    ESP-IDF 5.3.x project (pinned in CI container tag)
    CMakeLists.txt  sdkconfig.defaults  partitions.csv (reserves ota_0/ota_1, unused)
    Kconfig.projbuild                 TROPHIC_BOARD_*, TROPHIC_STATUS_LED_GPIO, TROPHIC_UART_*
    main/  app_main.c  boot.c
    components/board/                 board_<variant>.h + board.c (pin table: inputs + LED only)
    components/racklink/              framing (JSONL over UART), link_seq, t_mono_ms, 4 KB guard
    components/advert/                builds profile from detection + board descriptor
    components/sample_path/           register image: latest value per channel + seq + change flags
    components/drivers/sht/           I2C T/RH behind trh_driver_t
    components/drivers/contacts/      leak, LSH-04 dry contacts (GPIO input, debounce)
    components/drivers/tacho/         4× PCNT pulse counters → Hz
    components/sysaction/             sys.identify (LED), sys.ping
    components/diag/                  sys.heap_free (optional)
    test/host/                        host-side conformance test (cJSON, goldens)
    README.md                         relay hard gate (§7.1), bench rules (F-8)
  room-controller/README.md           "Empty until OQ-3 closes as ADR-0007 …" (§7.7)
```

### 1.4 `sim/` — Go module `github.com/trophic-corp/cea-orchestrator/sim` (fourth subsystem)

```
sim/
  go.mod go.sum (no require/replace on ../backend — boundary-check CI)
  Dockerfile
  cmd/virtual-room/main.go            the virtual room controller
  cmd/simctl/main.go                  CLI over the local control socket (§6.7)
  internal/vroom/                     controller.go, bridge.go, ring.go, health.go, cmdrouter.go
  internal/vdevice/                   device.go (profile → runtime), diurnal.go, channels.go
  internal/racklink/                  link.go (interface), inproc.go, serial.go, frame.go
  internal/faults/                    switches.go, override.go (test-only)
  internal/scenario/                  loader.go, runner.go (YAML timeline)
  internal/broker/                    mqtt.go (paho, mTLS, LWT, persistent session)
  internal/store/                     BufferStore: memory.go, file.go
  internal/clock/                     Clock: real.go, fake.go
  internal/schema/                    own payload structs + validation against docs/iot/schema/v1
  internal/ctl/                       simctl HTTP on unix socket / overlay-internal port
  internal/pki/                       simctl pki init (ephemeral CA, ECC P-256)
  profiles/rack-controller-std-v0.json room-controller-v0.json io-terrace-v0.json
           io-skid-v0.json io-room-v0.json esp32-skeleton-v0.json (captured)
  scenarios/72h-soak.yaml short-soak.yaml reject-path.yaml commissioning.yaml
            safety-override.yaml export-restore.yaml home-5s.yaml load-52-zones.yaml
  README.md                            "This is a simulator …" (IoT §5.2 wording verbatim)
```

### 1.5 Contracts of record and ops

```
docs/iot/schema/v1/                  README.md, envelope.schema.json, capability.schema.json,
                                     profile.schema.json, telemetry.schema.json, cmd.schema.json,
                                     ack.schema.json, health.schema.json, event.schema.json,
                                     state.schema.json, enums/{reason_code,sys_action,unit,
                                     event_type,clock,quality}.schema.json,
                                     racklink-frame.schema.json, rack-link-0.md,
                                     golden/<msg>/<name>.json (≥1 per type, per profile)
docs/api/openapi.v1.yaml             OpenAPI 3.1 (§4). (Task text said openapi.yaml; the HLD name is kept.)
docs/ops/restore-drill.md            runbook (§9.5)
docs/ops/sim-on-host.md              overlay teardown rule (F-2 (3))
ops/mosquitto/{Dockerfile,entrypoint.sh,mosquitto.conf}
ops/pki/{README.md,issue-device.sh,issue-service.sh,revoke.sh,.gitignore(out/)}
docker-compose.prod.yml  docker-compose.sim.yml
```

---

## 2. Wire schema v1 (`docs/iot/schema/v1/`) and `rack-link/0`

All schemas are JSON Schema **draft 2020-12**, `$id: https://trophic.example/iot/schema/v1/<name>`
(a URI namespace, not a URL that resolves), `additionalProperties: true` on message bodies
(forward compatibility, ADR-0002 rule 3) but **`false` on the envelope's enumerated fields'
types** — unknown *fields* are tolerated, wrong *types* are rejected.

### 2.1 Topic grammar and ACL patterns

```
trophic/<org>/<site>/<room>/<zone>/<device>/<leaf>
  <org>,<site>,<room>  [a-z0-9][a-z0-9-]{0,31}
  <zone>               rack-[0-9]{2} | room
  <device>             [a-z][a-z0-9-]{1,31}   (rc-01…, rmc-01, io-terrace, io-skid, io-room)
  <leaf>               capability | state/health | state/reported | state/desired |
                       telemetry/<metric> | cmd | ack | event
  <metric>             [a-z][a-z0-9_]{0,31}
```

`ingest/topic.go` parses with a strict regexp and rejects (drops, counts, logs) topics that do
not match — the only drop in the pipeline, since an unparseable topic has no `device_id` to
attach a row to. Retained: `capability`, `state/health`, `state/reported`, `state/desired`.
QoS 1 everywhere; the backend subscribes with `CleanSession=false`, client id `backend-ingest`.

**Mosquitto ACL patterns (materialised per principal, §5.8).** The aclfile format has no
deny rule, so the room-controller write set is **enumerated by leaf** rather than `#`; this
is how F-11 is satisfied (a room controller cannot publish `cmd` or `state/desired`):

```
user rmc-01
topic write trophic/<org>/<site>/<room>/+/+/capability
topic write trophic/<org>/<site>/<room>/+/+/state/health
topic write trophic/<org>/<site>/<room>/+/+/state/reported
topic write trophic/<org>/<site>/<room>/+/+/telemetry/+
topic write trophic/<org>/<site>/<room>/+/+/ack
topic write trophic/<org>/<site>/<room>/+/+/event
topic read  trophic/<org>/<site>/<room>/+/+/cmd

user backend-ingest
topic read  trophic/<org>/<site>/#
topic write trophic/<org>/<site>/+/+/+/cmd

# bench-direct ESP32 (bench CA only, never in prod aclfile): same leaves, prefix …/<zone>/<device>/
# harness (sim overlay only): read trophic/#, write trophic/<org>/<site>/+/+/+/cmd
```

Principal name = certificate CN = `device_id` (or service name). `pattern` lines are not used.

### 2.2 Envelope (`envelope.schema.json`, required on every message)

| Field | Type | Rule |
|---|---|---|
| `v` | integer const `1` | unknown → stored raw + `event{schema_unsupported}` (ingest ②) |
| `msg` | enum `capability,health,telemetry,cmd,ack,event,state` | |
| `org`,`site`,`room` | string, topic charset | must equal topic segments (else `topic_mismatch`) |
| `zone` | string `^(rack-[0-9]{2}\|room)$` | |
| `device_id` | string | must equal topic `<device>` (else `quality: suspect`, `topic_mismatch`, security audit row) |
| `boot_id` | string `^[a-z0-9]{6,16}$` | publisher's boot id |
| `seq` | integer ≥ 1 | per `(device_id, boot_id)`, assigned by the publisher |
| `ts_source` | RFC 3339 UTC, ms | publisher wall clock |
| `t_mono_ms` | integer ≥ 0, optional | from rack-link frame |
| `link_seq` | integer ≥ 1, optional | from rack-link frame |
| `clock` | enum `ntp,rtc,unsynced` | `unsynced` → `time_basis: ingest` |
| `sim` | boolean | fixed at claim |
| `backfill` | boolean | replayed from the ring |

### 2.3 Capability advert and profile (`capability.schema.json`, `profile.schema.json`)

```jsonc
{ "...envelope", "msg": "capability",
  "profile_sha256": "hex64",                    // sha256 of canonical JSON (RFC 8785) of "profile"
  "profile": {
    "schema_version": "1.0",
    "device_type": "rack-controller",           // descriptive only
    "hw_rev": "RK-A-E01 r2", "fw_version": "0.1.0",
    "hw_fingerprint": "24:6f:28:aa:bb:cc",      // eFuse MAC or virtual uuid; forwarded unaltered (F-14)
    "sim": true, "link": "rack-link/0" | "mqtt" | "in-process",
    "tiers": 4,                                 // racks only; drives instance defaults
    "capabilities": [ /* CapabilityInstance[] */ ],
    "commands":     [ { "class": "control.compute", "actions": ["sys.identify", "sys.ping"] } ]
  } }
```

`CapabilityInstance`:

```jsonc
{ "class": "sense.temp_rh",                     // ^[a-z]+\.[a-z_]+$ ; unknown → recorded, not rendered
  "instance": 0,                                // per class; tier index for per-tier classes
  "scope": "rack-01/tier-3" | "rack-01" | "room" | "terrace" | "skid" | "device",
  "tag": "LD-01" | null,                        // instrument tag from contracts or null (never invented)
  "hardware": "present" | "optional",
  "params": { ... },                            // class-specific; valve.control: {"failsafe_state":"CLOSED"|"OPEN", "duty":"fill"|"drain"}
                                                // control.compute: {"role":"rack"|"room"|"io-node","max_duration_s":60}
  "metrics": [ { "name": "air_temp_c", "unit": "degC", "range": [-40, 85], "range_note": "project-doc placeholder",
                 "resolution": 0.01, "accuracy": 0.3, "kind": "number"|"bool" } ],
  "interval_s": 30, "on_change": false,
  "role": "safety-mirror" | null                // sense.co2 only
}
```

Validation (`registry/profile.go`, mirrored in `sim/internal/schema` and the firmware host
test): (a) `commands[].class ⊆ {capabilities[].class}` — **no exemption**; (b) every
`sys.*` action ∈ `enums/sys_action` (`sys.identify`, `sys.ping`; `sys.backfill` present as
`reserved: true` and rejected); (c) `sys.*` actions only under `class: control.compute`;
(d) `unit` ∈ `enums/unit`; (e) `valve.control.params.failsafe_state` ∈ `{CLOSED, OPEN}`
(the leading token of `safety-rules.json valve_and_actuator_fail_safe_states.rack_fill_solenoids`
/ `.rack_drain_solenoids`); (f) `sense.co2` with `role: safety-mirror` **may omit `range`**;
(g) nothing in `absent-by-design.md` (class `sense.co2` at a rack scope, `sense.water_level` with
`kind: tray`, `sense.par` fixed, per-tier `sense.temp_rh` on a `std` profile) — (g) is a
*warning* on unknown devices and a *commissioning step-2 fault* on claimed ones.

`enums/unit`: `degC, pct, ppm, umol_mol, kPa, mS_cm, L_min, m3_h, m_s, Hz, bool, bytes`.
`bytes` (for `sys.heap_free`) joins `degC, pct, Hz, bool` on the contracts v0.3.0 review agenda.

### 2.4 Telemetry (`telemetry.schema.json`)

```jsonc
{ "...envelope", "msg": "telemetry",
  "metric": "air_temp_c", "value": 22.41 | true | null | "nan",   // type checked at ingest, not schema (never drop)
  "unit": "degC", "scope": "rack-01/tier-3", "tag": "LD-01" | null,
  "quality": "good" | "suspect", "quality_reason": null | string,   // device verdict; backend only downgrades
  "calibration_id": null }
```

### 2.5 Command and ack (`cmd.schema.json`, `ack.schema.json`)

```jsonc
// cmd
{ "...envelope(device_id = target, sim = target.sim, backfill = false)", "msg": "cmd",
  "cmd_id": "ULID (26 chars, Crockford)", "cmd_seq": 88,
  "capability": "control.compute", "action": "sys.identify",
  "params": { "duration_s": 10 },
  "issued_at": "…", "expires_at": "… (= issued_at + ttl_s)",
  "actor": { "kind": "user"|"system", "id": "u_01H…" } }

// ack
{ "...envelope", "msg": "ack", "cmd_id": "…",
  "result": "SUCCESS" | "REJECTED" | "TIMEOUT" | "EXPIRED",
  "reason_code": null | enums/reason_code, "reason": null | string,
  "duplicate": false, "latency_ms": 143,
  "state_after": { "identify": { "active": true, "until": "…" } } | { "ping": { "uptime_s": 1, "fw_version": "0.1.0", "clock": "unsynced", "seq": 1 } } }
```

`enums/reason_code` (closed): `capability_not_advertised, action_not_supported,
params_out_of_range, schema_version_unsupported, device_unclaimed, stale_sequence, expired,
busy` (Phase 1) + reserved `interlock_active, envelope_violation, e_stop, not_in_window`
(schema-listed with `"reserved": true`; edge and backend treat them as valid enum members
that no Phase 1 path emits).

### 2.6 Health (`health.schema.json`) and event (`event.schema.json`)

Health as IoT §1.6 verbatim plus `identify: {active, until}` (so the console can show the
reported identify state). `status ∈ {online, degraded, offline}`; `reason` for offline ∈
`{lwt, shutdown, bus_timeout}`; for degraded ∈ `{clock_unsynced, buffer_dropped,
sensor_init_failed, command_timeout}`. LWT payload is a full valid health message with
`status: offline, reason: lwt, seq: 0` (seq 0 is reserved for LWT and excluded from gap
detection).

Event: `{ "...envelope", "msg": "event", "type": enums/event_type, "detail": {...} }` with
`event_type` (wire, closed): `boot, claim, link_gap {device_id, link_seq_from, link_seq_to},
buffer_overflow {seq_from, seq_to}, schema_unsupported, clock_anomaly, unit_unexpected,
link_frame_invalid {reason}, profile_changed, identify {active}`. Backend-only kinds stored
in `device_events` but never on the wire: `topic_mismatch, sim_flip, fingerprint_mismatch,
quality_transition, vpd_unpaired, ingest_outage`.

### 2.7 `rack-link/0` frame (`racklink-frame.schema.json`, `rack-link-0.md`)

One UTF-8 JSON object per line, `\n` terminated, **≤ 4096 bytes including the newline**
(brief D-2 / §10.1 C(a): the 4 KB limit applies to this frame, not to the MQTT message):

```jsonc
{ "rl": 0, "link_seq": 4711, "t_mono_ms": 812934, "msg": "telemetry"|"capability"|"health"|"ack"|"event",
  "device_id": "rc-bench-01", "body": { /* v1 payload object WITHOUT org/site/room/ts_source/clock/seq/boot_id/sim/backfill */ } }
// downstream (room → rack): { "rl": 0, "msg": "cmd", "device_id": "rc-bench-01", "body": { cmd_id, cmd_seq, capability, action, params, expires_at } }
```

Receiver rules (`sim/internal/racklink/frame.go`; firmware `components/racklink`): a line
over 4096 bytes is discarded at the byte counter (not parsed) and the room controller
publishes `event{link_frame_invalid, reason: oversize}` for that `device_id` if known;
non-JSON or missing `rl/msg/device_id` → `event{link_frame_invalid, reason: parse}`;
`link_seq` jump > 1 → `event{link_gap}` and a `telemetry_gaps` row `unfillable{link_gap}` at
the backend; `t_mono_ms` regression → the room controller treats it as a rack reboot
(new rack boot) and records `event{boot}` on the rack's behalf. The room controller stamps
`org, site, room, zone, ts_source, clock, seq, boot_id, sim, backfill` and publishes.
**Security statement (F-14), verbatim in `rack-link-0.md`:** "`rack-link/0` has no
authentication, no integrity protection and no replay protection; the virtual room
controller trusts whatever is on the serial adapter. This is acceptable only because Phase 1
has no actuation and the adapter is a bench USB-serial link; the R2 Modbus map (contracts
v0.3.0) and the two-party commissioning fingerprint attestation must close before a
bus-attached device is trusted for anything that moves." Sunset: replaced by the Modbus
register map in R2; ADR-0007 must reference this file.

Goldens: `golden/capability/{rack-std,room,io-terrace,io-skid,io-room,esp32-skeleton}.json`,
`golden/telemetry/{air_temp_c,air_rh_pct,leak,header_high,fan_tacho_hz,co2_ppm,water_temp_c,ph,ec_ms_cm,level_pct,heap_free}.json`,
`golden/cmd/sys-identify.json`, `golden/ack/{success,rejected-capability_not_advertised,
rejected-action_not_supported,rejected-params_out_of_range,duplicate,timeout,expired}.json`,
`golden/health/{online,lwt,bus_timeout,degraded-command_timeout}.json`, `golden/event/*.json`,
`golden/racklink/{telemetry,capability,cmd,ack}.jsonl`. The `schema` CI job validates each
against its schema with `ajv-cli` (no subsystem code); each subsystem's conformance test
parses and re-emits them byte-equal after canonicalisation.

---

## 3. Database (PostgreSQL 16 + TimescaleDB 2.17)

### 3.1 Migration tool and roles

**goose v3, SQL migrations, embedded (`//go:embed migrations/*.sql`), forward-only**: every
file has `-- +goose Up` only; a `-- +goose Down` section fails a repo lint. Version is stored
in `goose_db_version`; `backend migrate` prints the version and the bundle manifest records
it as `db_schema_version`. Two roles: `trophic_migrate` (owner; used only by `migrate`,
`seed`, `restore`) and `trophic_app` (runtime; `SELECT/INSERT/UPDATE/DELETE` on all tables
**except** `audit_log` = `SELECT, INSERT` only). Both passwords are compose secrets
(`db_password`, `db_migrate_password`).

Conventions: every table has `org_id uuid NOT NULL REFERENCES orgs(id)` as the **first**
column after `id`; primary keys are `uuid` (v7 generated in Go); all timestamps
`timestamptz`; enums are `text` + `CHECK` (cheap to extend forward-only). **Scoping is
repository-layer (ADR-0005; RLS deferred)** — §5.9 gives the gate. Hypertables carry `org_id`
in every index leading position.

### 3.2 Tenancy and facility

```sql
orgs(id uuid pk, slug text unique, name text, created_at)
sites(id, org_id, slug, name, tz text default 'Asia/Kolkata', unique(org_id, slug))
rooms(id, org_id, site_id fk, slug, name, lifecycle text check in (installed,active,maintenance,retired), unique(org_id, site_id, slug))
racks(id, org_id, room_id fk, slug 'rack-01', name, lifecycle, position_ref text, notes text, doc_links jsonb, tiers int not null check (tiers between 1 and 8), unique(org_id, room_id, slug))
tiers(id, org_id, rack_id fk, index int, tier_height_mm int, notes, doc_links jsonb,
      commissioning_ppfd numeric, commissioning_ppfd_entered_by uuid, commissioning_ppfd_entered_at timestamptz, unique(org_id, rack_id, index))
zones(id, org_id, room_id fk, slug 'Z-1-3', name, kind text check in (tier,room,terrace,skid),
      status text check in (active,idle), status_reason text, status_set_by uuid, status_set_at,
      derived_offline bool default false, derived_offline_reason text, unique(org_id, room_id, slug))
zone_tiers(org_id, zone_id fk, tier_id fk, pk(zone_id, tier_id))
```

`GET /facility/tree` reads `zones.status` and `derived_offline` and reports
`zone_status = offline if derived_offline else status` (FR-FAC-03); clearing `derived_offline`
restores the operator-set `status` automatically because it is a separate column.

### 3.3 Identity

```sql
users(id, org_id, email citext, display_name, role text check in (admin,grower,operator,technician,viewer),
      password_hash text /* argon2id PHC string; t=3, m=64MiB, p=4 */, disabled bool, created_at, last_login_at, unique(org_id, email))
totp_secrets(user_id pk fk, org_id, secret_ct bytea /* AES-256-GCM under totp_key; nonce||ct||tag */, key_version int, enrolled_at, last_used_step bigint /* replay guard */)
totp_enrolments(id, org_id, user_id, token_hash bytea, expires_at, used_at)   -- one-time enrolment links (admin-issued)
sessions(id_hash bytea pk /* sha256(opaque 32-byte id) */, org_id, user_id, created_at, last_seen_at, idle_expires_at, absolute_expires_at, revoked_at, ip inet, ua text)
login_attempts(org_id, email citext, at, ok bool)   -- rate limit source; pruned hourly
```

Sessions are server-side opaque ids; **no signing key exists** (F-12). The secret set is
therefore: `totp_key` (32 bytes; travels with backups, §5.7) and `backup_key` (age identity);
`session_key` from HLD §7.3 is **removed**. `sessions`, `login_attempts`, `totp_enrolments`
are **excluded from the bundle** (F-3) and declared in `manifest.excluded_tables`.

### 3.4 Registry

```sql
devices(id, org_id, device_id text /* topic segment */, parent_device_id text null, device_type text, hw_rev, fw_version,
        lifecycle text check in (unclaimed,claimed,commissioned,active,retired,quarantined),
        lifecycle_reason text, hw_fingerprint text, sim bool null /* fixed at claim */,
        credential_type text check in (mtls-ecc-p256,none), credential_fingerprint text null, credential_serial text null, acl_prefix text null,
        profile_sha256 text, profile_id uuid fk device_profiles, label text, first_seen_at, last_seen_at,
        claimed_by uuid, claimed_at, activated_at, retired_at, unique(org_id, device_id))
device_profiles(id, org_id, device_id, profile_sha256, schema_version text, profile jsonb, received_at, unique(org_id, device_id, profile_sha256))   -- append-only per advert change
capability_index(id, org_id, device_id, class, instance int, scope, tag, hardware, params jsonb, metrics jsonb, interval_s int, role text null,
                 in_commands bool, actions text[], recognised bool, unique(org_id, device_id, class, instance))
device_assignments(id, org_id, device_id, class, instance, zone_id fk, tier_id fk null, assigned_by, assigned_at, unique(org_id, device_id, class, instance))
commissioning_steps(id, org_id, device_id, step int check between 0 and 5, outcome text, detail jsonb, actor_user_id, confirmed_by_name text null, at)   -- append-only (trigger, §3.9)
identify_confirmations(id, org_id, device_id, cmd_id text, seen bool, confirmed_by_user_id, confirmed_by_name text, at)
device_health_current(org_id, device_id pk, connectivity text check in (unknown,online,offline), offline_reason text, offline_inherited bool,
        degraded_reason text null, fault bool, badge text check in (online,offline,fault,degraded,stale_telemetry),
        last_heartbeat_ingest timestamptz, last_telemetry_ingest timestamptz, stale_metrics text[], updated_at)
class_fallback_ranges(class, metric, lo numeric, hi numeric, note text, pk(class, metric))   -- seeded from §11 table; every row carries its label
```

### 3.5 Telemetry hypertables, rollups, retention

```sql
telemetry_raw(org_id, device_id, metric, scope, tag, ts_effective timestamptz not null, ts_source timestamptz, ts_ingest timestamptz not null,
   time_basis text check in (source,ingest), seq bigint, boot_id text, value double precision null, value_bool bool null, value_raw text null,
   unit text, quality text check in (good,suspect,implausible-rejected), quality_reason text, range_source text check in (advert,class_fallback,none),
   calibration_id text, backfill bool, unclaimed bool, zone_id uuid null)
 -> create_hypertable(by ts_ingest, chunk 1 day); index (org_id, device_id, metric, ts_ingest desc); index (org_id, zone_id, metric, ts_effective desc) where zone_id is not null
 -> compression: segmentby (org_id, device_id, metric), orderby ts_ingest; add_compression_policy('7 days')
 -> add_retention_policy('24 months')   -- ops-configurable via `backend retention --raw 24mo` (writes the policy; migration never hardcodes a shorter value)
telemetry_dedup(org_id, device_id, boot_id, seq, seen_at, pk(org_id, device_id, boot_id, seq))   -- plain table, pruned > 48 h (2 × ring 6 h × margin)
telemetry_derived(org_id, device_id, zone_id, metric text /* vpd_kpa | dli_mol_m2_day */, ts_effective, ts_ingest, value, unit, formula_version text,
   provenance text check in (calculated,estimated), quality, quality_reason, inputs jsonb /* [{metric, ts_source, quality, accuracy}] */, leaf_temp_assumption text, backfill bool)
 -> hypertable by ts_ingest; same policies as raw
device_events(org_id, device_id, ts_ingest, kind text, detail jsonb, security bool default false) -> hypertable, retention none
device_health_events(org_id, device_id, at, from_state text, to_state text, reason text, inherited bool) -> hypertable
last_known_state(org_id, device_id, metric, scope, pk(org_id, device_id, metric, scope), zone_id, tag, value, value_bool, unit, ts_source, ts_ingest, time_basis,
   quality, quality_reason, range_source, calibration_id, provenance text, formula_version text null, inputs jsonb null, accuracy numeric null,
   staleness_max_age_s int, sim bool, updated_at)   -- written only by live, non-backfill, non-rejected rows (ingest ⑨)
telemetry_gaps(id, org_id, device_id, boot_id, seq_from bigint, seq_to bigint, ts_from, ts_to, detected_at, status text check in (open,filled,unfillable),
   cause text check in (backend-down,device-offline,broker-down,buffer_overflow,link_gap,not_buffered,unknown), reason text, filled_at, filled_count int)
ingest_outages(id, org_id, started_at, ended_at, kind text check in (startup,broker-disconnect), note)
platform_health(org_id, at, ingest_lag_s numeric, broker_connected bool, storage_bytes bigint, restart_marker bool, sim_overlay bool) -> hypertable, retention '90 days'
```

Continuous aggregates (`FR-TEL-05`): `telemetry_1m`, `telemetry_1h`, `telemetry_1d` over
`telemetry_raw` grouped by `(org_id, zone_id, device_id, metric, bucket)` with `min, max,
avg, count, n_suspect, n_rejected`, **`avg/min/max` computed over `quality <>
'implausible-rejected'`**, `n_suspect` counted; `telemetry_derived` gets the same three.
Refresh policies: 1m every 1 min (start offset 2 h), 1h every 10 min, 1d every 1 h.
Real-time aggregation on for 1m. Chart cut-over (`resolution=auto`): range ≤ 6 h → raw;
≤ 7 d → 1m; else 1h. `raw` may be requested explicitly for ≤ 24 h only (413 otherwise).

### 3.6 Alerts

```sql
alert_rules(id, org_id, key text unique(org_id,key) /* e.g. climate.air_temp_band */, name, description text,
   metric text, class text, scope_kind text check in (zone,rack,room,device,facility), role_filter text null /* safety-mirror */,
   comparator text check in ('>','>=','<','<=','outside','==','is_true'), threshold_lo numeric null, threshold_hi numeric null, threshold_bool bool null, unit text,
   duration_s int not null, clear_s int not null, hysteresis numeric default 0, hysteresis_note text,
   severity text check in (SAFETY,OPERATIONAL,ADVISORY), enabled bool, is_interlock_mirror bool,
   source_citation jsonb not null /* {kind: safety_rules|contracts|design_parameter|operator, key_path, source_string, comparator_basis} */,
   protects_text text, hardware_state_text text, action_codes text[] not null default '{}', editable bool, duration_label text /* 'unsourced-default' */,
   renotify_s int null)
   CHECK (severity <> 'SAFETY' OR (duration_s = 0 AND editable = false AND enabled = true AND array_length(action_codes,1) >= 1))
   CHECK (is_interlock_mirror = false OR severity = 'SAFETY')
alert_rule_overrides(id, org_id, rule_id fk, zone_id fk, threshold_lo, threshold_hi, duration_s, clear_s, enabled, label text default 'operator-configured, unsourced', set_by, set_at, unique(org_id, rule_id, zone_id))
alert_eval_state(org_id, rule_id, scope_id text /* zone/device/room id */, pk(org_id, rule_id, scope_id),
   state text check in (idle,breaching,raised,clearing), since timestamptz, alert_id uuid null, last_value numeric, last_value_bool bool,
   last_sample_ts timestamptz, last_sample_quality text, monitor_silent_since timestamptz null, updated_at)
alerts(id, org_id, rule_id, scope_id, scope_path text, severity, state text check in (raised,acknowledged,resolved),
   raised_at, notified_at, ack_by uuid null, ack_at, ack_action_code text, ack_note text,
   resolved_at, resolved_by uuid null /* null = auto */, resolve_reason text, resolve_kind text check in (auto,operator),
   value_snapshot jsonb /* value envelope at raise */, current_snapshot jsonb, affected_zones uuid[], suppressed_under uuid null,
   is_test_injection bool default false, monitor_silent_since timestamptz null, renotify_count int)
   index (org_id, state, severity, raised_at desc)
alert_notifications(id, org_id, alert_id, channel text check in (console,email), target text, sent_at, ok bool, error text)
notification_channels(id, org_id, severity, channel, target text, enabled, created_by, unique(org_id, severity, channel, target))
   -- rows with severity='SAFETY' and channel='console' are seeded and undeletable (trigger raises)
```

### 3.7 Seeded rules — verbatim thresholds, comparator recorded

`alerts/seed.go` inserts these idempotently (`ON CONFLICT (org_id, key) DO UPDATE` only for
non-SAFETY description text; SAFETY rows are never updated by a re-seed, only inserted).
`source_citation.source_string` is the `source` field of the cited key, copied byte-for-byte.
`comparator_basis` quotes the wording that fixed the comparator.

| key | metric / class (role) | comparator, thresholds | sev | dur / clear (s) | citation `key_path` |
|---|---|---|---|---|---|
| `climate.air_temp_band` | `air_temp_c` / `sense.temp_rh` @ zone(rack) | `outside [20, 24]` | ADVISORY | 300 / 300 `unsourced-default` | `climate.air_temperature_c` |
| `climate.rh_band` | `air_rh_pct` | `outside [55, 70]`, hysteresis 5 (`hysteresis_note`: "borrowed from control: `climate.relative_humidity_pct.hysteresis_pct`") | ADVISORY | 300 / 300 | `climate.relative_humidity_pct` |
| `climate.vpd_band` | `vpd_kpa` (derived) | `outside [0.6, 1.0]` | ADVISORY | 300 / 300 | `climate.vpd_kpa`; description carries botany §2.3 text |
| `climate.co2_enrichment_band` | `co2_ppm` / `sense.co2` role null @ room | `outside [1000, 1500]` unit `umol_mol` | ADVISORY, **enabled=false**, description "enable when CO2 enrichment is commissioned (R2)" | 300 / 300 | `climate.co2_enrichment_target_umol_mol` |
| `water.solution_temp_operating_max` | `water_temp_c` / `sense.water_temp` (TE-01, TE-02) | `> 26` (`comparator_basis`: "above 26C" note; "alarm at 26 °C" contracts) | OPERATIONAL | 120 / 120 | `irrigation_fertigation.solution_temperature_c.operating_max` |
| `water.solution_temp_hard_inhibit_mirror` | `water_temp_c` | **`> 28`** (`comparator_basis`: "exceeds 28C"; contracts `> 28 °C`) | **SAFETY**, mirror | **0** / 120 | `…solution_temperature_c.hard_inhibit_c`; `hardware_state_text` = the key's `action` string verbatim |
| `water.ph_target_band` | `ph` / `sense.ph` (AT-04; TK-01 pH untagged) | `outside [5.6, 6.2]` | ADVISORY | 600 / 600 | `irrigation_fertigation.ph_operating_target` |
| `safety.co2_alarm_mirror` | `co2_ppm` / `sense.co2` **role = safety-mirror only** | **`>= 5000`** (`comparator_basis`: "Alarm at 5000 ppm") | **SAFETY**, mirror | **0** / 120 | `climate.co2_safety_alarm_ppm`; `protects_text` = `climate.co2_asphyxiation_context` verbatim; `hardware_state_text` = "independent monitor, fail-closed solenoid" |
| `safety.leak_puck_mirror` | `leak` / `sense.leak` (LD-NN) | `is_true` | **SAFETY**, mirror | 0 / 120 | contracts `safety/interlocks.md` row "Rack leak puck" |
| `safety.lsh04_mirror` | `header_high` / `sense.water_level{kind: switch}` | `is_true` | **SAFETY**, mirror | 0 / 120 | contracts `interlocks.md` row "Header high-level switch LSH-04" |
| `safety.estop_mirror`, `safety.floor_flood_mirror` | `estop_engaged`, `floor_flood` on `io-room` **only if advertised**; seeded `enabled=true` but evaluate against nothing until the class appears | `is_true` | **SAFETY**, mirror | 0 / 120 | contracts `interlocks.md` rows "E-stop", "Room floor flood sensor" |
| `device.fan_stopped` | `fan_tacho_hz` / `airflow.tacho` @ zone(tier) | `== 0` | OPERATIONAL | 90 / 90 | contracts `instrument-tags.md` "the tacho alarm is not optional"; Rack A §05 |
| `device.stale_telemetry` | per device, health axis | badge = `stale_telemetry` | OPERATIONAL | 0 / 0 (the max_age is the duration) | `design_parameter` "HLD §10.2 / LLD §11" |
| `device.offline` | per device | connectivity = offline, not inherited | OPERATIONAL | 0 / 0 (`offline_after_s` is the duration) | `design_parameter` |
| `device.telemetry_gap` | per device | any `telemetry_gaps.status = open` | OPERATIONAL | 0 / 0 | `design_parameter` |
| `platform.broker_disconnected` | facility | ingest client disconnected > 60 s | OPERATIONAL | 60 / 30 | `design_parameter` |
| `platform.backup_failed` | facility | last scheduled run status ≠ ok | OPERATIONAL | 0 / 0 | `design_parameter` |

Action codes: `safety.co2_alarm_mirror` → `[ventilated_and_left, checked_cylinder_line,
evacuated_room, other]`; `water.solution_temp_hard_inhibit_mirror` → `[checked_te_probe,
stopped_transfer, other]`; leak/LSH-04/flood → `[inspected_rack, cleared_drain, other]`;
E-stop → `[confirmed_estop_reason, other]`. `other` requires a non-empty `note`.

**Seed-time invariants (hard errors):** (i) for every rule with `is_interlock_mirror` and a
numeric threshold, every currently-known effective range (advert or fallback) for that
channel has `hi > threshold` — checked at seed and on every profile change
(`registry/profile.go` calls `alerts.CheckBoundInvariant`); (ii) every SAFETY row satisfies
the `CHECK` in §3.6; (iii) `duration_s ≥ 2 × min(interval_s)` for band rules (300 ≥ 60).
**Not seeded:** EC band, G1/G2, DLI, tacho-low, solution-T low, CO2 floor, divergence
tolerance, rate-of-change, fail-safe/electrical entries, saffron, aquatic.

### 3.8 Commands, audit, backup

```sql
commands(id uuid, org_id, cmd_id text unique(org_id, cmd_id), device_id, capability, action, params jsonb, actor_user_id, issued_at, expires_at, cmd_seq bigint,
   backend_state text check in (issued,acked,unacknowledged), ack_result text, ack_reason_code, ack_reason, ack_latency_ms int, ack_state_after jsonb, ack_received_at, duplicate bool, context text check in (commissioning,device_detail))
command_seq(org_id, device_id, pk, next bigint)   -- cmd_seq allocator
audit_log(id uuid, org_id, at, actor_kind text check in (user,system,device), actor_id text, category text check in (authn,alert,backup,user,device,config,zone,rule,command,security), action text,
   target_kind, target_id, before jsonb, after jsonb, request_id text, detail jsonb)
   -- append-only: trophic_app has SELECT,INSERT only; trigger audit_log_immutable BEFORE UPDATE OR DELETE FOR EACH ROW EXECUTE raise_exception('audit_log is append-only')
   -- the trigger is also installed on commissioning_steps and device_profiles
backup_runs(id, org_id, kind text check in (scheduled,on-demand), started_at, finished_at, status text check in (running,ok,failed), bundle_path, size_bytes, sha256, manifest jsonb, error, initiated_by uuid null)
import_runs(id, org_id, started_at, finished_at, bundle_sha256, policy text check in (validate,import-as-new,replace), report jsonb, status, initiated_by text /* CLI operator string */)
```

AC-USR-02a tests: as `trophic_app`, `UPDATE audit_log` → permission denied; as
`trophic_migrate`, `UPDATE audit_log` → trigger exception. Both must fail.

### 3.9 Seed file (`backend/seeds/facility-ooty-r1.v1.yaml`)

`org: {slug: trophic, name}` → `site: {slug: ooty-r1, tz}` → `rooms[room-1]` → `racks[rack-01..rack-11, tiers: 4]`
→ tiers auto → `zones` one per tier (`Z-<rack>-<tier>`, kind tier) + `room`, `terrace`, `skid`
zones. No X/Y coordinates (OQ-4). `seed_version: 1` recorded in `orgs.seed_version`; a re-run
with the same version is a no-op; a newer version adds only (never deletes). The seed also
runs `alerts.Seed` and `class_fallback_ranges`, and seeds `notification_channels
(SAFETY, console)` and `(OPERATIONAL, console)`, `(ADVISORY, console)`. Users are **not**
seeded except via `backend seed --admin-email` which creates one admin with a TOTP enrolment
link printed once to stdout.

---

## 4. OpenAPI v1 (`docs/api/openapi.v1.yaml`)

Base path `/api/v1`; JSON only; cookie auth `trophic_session` (`HttpOnly; Secure;
SameSite=Strict; Path=/api`); CSRF: `SameSite=Strict` + required header
`X-Requested-With: trophic-console` on every non-GET (checked by middleware). Every 2xx
response body has top-level `as_of` (RFC 3339, server time) and the header `X-As-Of`.
Every list is cursor-paginated: `?limit=50&cursor=…` → `{items: [...], next_cursor: string|null, as_of}`.
Errors are RFC 9457 `application/problem+json`: `{type: "urn:trophic:problem:<code>", title,
status, detail, instance: request_id, code, fields?: {name: reason}}`. Codes used:
`validation_failed 400`, `unauthenticated 401`, `forbidden 403`, `not_found 404`,
`conflict 409` (lifecycle/claim/assignment), `precondition_failed 412` (resolve refused,
ack without action code), `rate_limited 429`, `unsupported_class 400` (command class ∉
allow-list), `range_too_large 413`. **No endpoint accepts an org id**; `org` is resolved from
the session and never appears in a path.

### 4.1 Cross-cutting schemas

```yaml
ValueEnvelope:            # every quantitative field, no exceptions
  required: [value, unit, provenance, ts_source, time_basis, age_s, stale, staleness_max_age_s, quality, sim]
  properties:
    value: {oneOf: [number, boolean, "null"]}     # null when no admitted sample exists
    unit: {$ref: Unit}
    provenance: {enum: [measured, calculated, estimated, operator_entered]}
    ts_source: {format: date-time, nullable}   time_basis: {enum: [source, ingest]}
    age_s: {type: number}                       # as_of − ts_effective; ts_effective = ts_source if time_basis=source else ts_ingest
    stale: {type: boolean}                      # ts_ingest verdict (D-12)
    staleness_max_age_s: {type: integer}
    quality: {enum: [good, suspect, implausible_rejected]}   quality_reason: {string, nullable}
    formula_version: {string, nullable}   inputs: {array of {metric, ts_source, quality, age_s, accuracy}}   leaf_temp_assumption?
    calibration_id: {string, nullable}   sim: {boolean}   instrument_tag: {string, nullable}
    range_source: {enum: [advert, class_fallback, none]}   accuracy: {number, nullable}
    entered_by: {string, nullable}   entered_at: {date-time, nullable}    # operator_entered only
    is_test_injection: {boolean, default false}
HealthBadge: {enum: [online, offline, fault, degraded, stale_telemetry, unknown]}
ClaimState:  {enum: [unclaimed, claimed, commissioned, active, retired, quarantined]}
Severity:    {enum: [SAFETY, OPERATIONAL, ADVISORY]}
Problem:     (RFC 9457 as above)
Page[T]:     {items: T[], next_cursor, as_of}
```

### 4.2 Endpoint list

| Method path | Roles | Request → Response (schema names) | Notes |
|---|---|---|---|
| `POST /auth/login` | anon | `LoginRequest{email,password,totp}` → `Session` (sets cookie) | 401 `unauthenticated` always the same body; rate limit 5/min/email + 20/min/ip (429) |
| `POST /auth/logout` | any | → 204 | revokes session |
| `GET /session` | any | → `Session{user_id, display_name, role, org_slug, session_expires_at, absolute_expires_at, safety_active_count}` | `safety_active_count` also served to the unauthenticated login shell via `GET /public/safety-banner` → `{active: int, room_names[]}` (text only, no ids) |
| `POST /auth/totp/enrol` | anon w/ token | `{token, code}` → 204 | completes admin-issued enrolment |
| `GET /facility/summary` | any | → `FacilitySummary` (§4.3) | the Home call; served from `last_known_state`, `alerts`, `device_health_current`, `platform_health` only |
| `GET /facility/tree` | any | → `FacilityTree{rooms[{id,slug,name,lifecycle,racks[{id,slug,name,lifecycle,position_ref,sim,tiers[{id,index,zone_id}],zone_status_counts,alert_counts,controller_device_id,health}],zones[…]}]}` | |
| `GET /rooms/{id}/snapshot` | any | → `RoomSnapshot{room, metrics: {<key>: ValueEnvelope}, water: {<tag>: ValueEnvelope}, racks: RackRow[]}` | metric key = `<class>/<scope>/<instance>/<metric>`; only advertised metrics appear |
| `GET /racks/{id}/snapshot` | any | → `RackSnapshot{rack, controller: DeviceSummary, env: {air_temp_c, air_rh_pct, vpd_kpa}: ValueEnvelope, interlocks: {leak, header_high}: ValueEnvelope, tiers[{tier, zone, status, reports: {fan_tacho_hz, light_state}: ValueEnvelope}], alerts: AlertRow[], events: EventRow[]}` | |
| `GET /zones/{id}/snapshot` | any | → `ZoneSnapshot{zone, rack_env, reports, capabilities: CapabilityInstance[] (union), devices: DeviceSummary[], dli: ValueEnvelope|null}` | `dli` null when no PPFD |
| `GET /zones/{id}/telemetry?metric=&from=&to=&resolution=auto\|raw\|1m\|1h` | any | → `Series{metric, unit, resolution_used, points[{ts, value, quality, calibration_id, backfill}] | buckets[{ts, min,max,avg,count,n_suspect,n_rejected}], gaps[{from,to,status,cause}], sim}` | 413 if raw > 24 h |
| `GET /zones/{id}/events?since=&cursor=` | any | → `Page[EventRow{ts, kind, device_id, detail}]` | |
| `GET /devices?health=&claim_state=&cursor=` | any | → `Page[DeviceSummary{id, device_id, label, device_type, assignment{room,rack,tier}?, health: HealthBadge, offline_inherited, last_seen_at, age_s, fw_version, sim, claim_state, lifecycle_reason, hw_fingerprint_last4, profile_schema_version, first_seen_at, parent_device_id, commissioning_step}]` | unclaimed rows carry `hw_fingerprint_last4` only |
| `GET /devices/{id}` | any | → `DeviceDetail` (summary + identity, claimed_by/at, profile_sha256, assignment[], commissioning_steps[], health{last_telemetry_at, last_event_at, staleness_by_metric{metric: max_age_s}, seq_gap_count, degraded_reason, identify{active, until}}, credential{type, fingerprint_last8})` | never a secret |
| `GET /devices/{id}/capabilities` | any | → `Capabilities{profile_sha256, schema_version, capabilities: CapabilityInstance[] + `in_commands`, `failsafe_check: {status: ok\|mismatch\|n/a, expected, advertised, source_key_path}` per valve.control, commands[], unrecognised[{class, raw}], expected_counts{from: "contracts instrument-tags.md v0.2.0", fill: 4, drain: 4, fans: 4, led_pairs: 4, trh: 1}, count_mismatches[]}` | render rule 3 of ADR-0010: `in_commands` is information only in v0 |
| `GET /devices/{id}/commands?cursor=` | any | → `Page[CommandRow{cmd_id, capability, action, params, issued_by, issued_at, expires_at, backend_state, ack{result, reason_code, reason, latency_ms, state_after, received_at, duplicate}?, confirmation{seen, confirmed_by, at}?}]` | |
| `GET /devices/{id}/events?cursor=` | any | → `Page[EventRow]` | |
| `POST /devices/{id}/claim` | technician, admin | `{fingerprint_confirm: 4 chars, label}` → `DeviceDetail` | 409 not unclaimed; 409 `parent_unclaimed`; 412 fingerprint mismatch (→ quarantined if identity known) |
| `POST /devices/{id}/commissioning/verify` | technician, admin | `{accepted: true, notes}` → `DeviceDetail` | 412 while `failsafe_check.status = mismatch` (blocking fault) |
| `POST /devices/{id}/assign` | technician, admin | `{rack_id \| zone_id, instance_map[{class, instance, tier_index}]}` → `DeviceDetail` | 409 rack already has a controller (`conflict_device_id`) |
| `POST /devices/{id}/commands` | technician, admin | `{action: "sys.identify", params{duration_s ≤ advertised max}}` → `CommandRow` 202 | 400 `unsupported_class` for anything else; 409 unless lifecycle ∈ {commissioned, active}; 412 params > max; 409 `busy` if one in flight |
| `GET /commands/{cmd_id}` | any | → `CommandRow` | poll target |
| `POST /devices/{id}/commissioning/confirm-identify` | technician, admin | `{cmd_id, seen: bool, confirmed_by_name?}` → `DeviceDetail` | records `identify_confirmations`; `seen=false` keeps step 4 |
| `POST /devices/{id}/activate` | technician, admin | `{zone_status: active\|idle, reason?}` → `DeviceDetail` | 412 unless a SUCCESS ack + `seen=true` confirmation exist for the current assignment |
| `POST /devices/{id}/retire` | technician, admin | `{reason}` → `DeviceDetail` | |
| `POST /zones/{id}/status` | admin, grower, operator | `{status: active\|idle, reason}` → `Zone` | audited |
| `GET /alerts?state=active\|acknowledged\|resolved&severity=&rule_id=&scope=&from=&to=&cursor=` | any | → `Page[AlertRow{id, severity, rule_id, rule_key, rule_name, scope_id, scope_path, raised_at, duration_s, state, current_value: ValueEnvelope, threshold{comparator, lo, hi, bool, unit}, ack{by, at, action_code, note}?, is_interlock_mirror, hardware_state_text?, affected_zones[], suppressed_under?, is_test_injection, monitor_silent_since?, resolvable, resolve_blocked_reason?}]` | `state=active` = raised ∪ acknowledged; sorted severity then raised_at |
| `GET /alerts/{id}` | any | → `AlertDetail` = AlertRow + `timeline[{at, event, actor?, detail}]`, `rule: RuleView`, `resolved{at, by, kind, reason}?` | |
| `POST /alerts/{id}/ack` | admin, grower, operator | `{action_code?, note?}` → `AlertDetail` | SAFETY: 412 `action_code_required` if missing or ∉ rule.action_codes; `other` needs note |
| `POST /alerts/{id}/resolve` | admin, grower, operator | `{reason}` → `AlertDetail` | 412 `condition_present` while eval state ≠ clearing/idle; SAFETY additionally 412 `monitor_silent` if no admitted in-band sample within `clear_s` |
| `GET /alert-rules?cursor=` `GET /alert-rules/{id}` | any | → `Page[RuleView{id, key, name, description, severity, scope_kind, metric, class, role_filter, comparator, threshold{lo,hi,bool,unit}, duration_s, clear_s, hysteresis, hysteresis_note, duration_label, enabled, editable, is_interlock_mirror, source_citation, protects_text, hardware_state_text, action_codes[], overrides[{zone_id, lo, hi, duration_s, clear_s, enabled, label}], noisy{acks_without_action_30d, auto_resolved_under_5min_30d, tagged_noisy}}]` | |
| `PATCH /alert-rules/{id}` | admin, grower | `{zone_id, threshold_lo?, threshold_hi?, duration_s?, clear_s?, enabled?}` → `RuleView` | **403 `safety_rule_immutable`** for `severity = SAFETY` or `is_interlock_mirror` on any field (F-6); creates/updates an override row only, never the rule |
| `GET /users` `POST /users` `PATCH /users/{id}` `POST /users/{id}/totp/reset` | admin | `UserCreate{email, display_name, role}` → `User + enrolment{url, expires_at}` (shown once); `UserPatch{role?, disabled?}` | admin cannot disable self |
| `GET /notification-channels` `POST /notification-channels` `DELETE /notification-channels/{id}` | admin | `{severity, channel: email, target}` | DELETE 403 `safety_channel_additive_only` when severity = SAFETY |
| `GET /backups` | admin | → `{schedule{cron, retention}, last_run: BackupRun, runs: Page[BackupRun{id, kind, started_at, finished_at, status, size_bytes, sha256, manifest_summary, error}]}` | |
| `POST /backups/export` | admin | `{raw_window_days?}` → `BackupRun` 202 | audited |
| `GET /backups/jobs/{id}` | admin | → `BackupRun` | |
| `GET /backups/{id}/download` | admin | → `application/octet-stream` | audited |
| `POST /backups/import/validate` | admin | multipart bundle → `ValidationReport{bundle_schema_version, db_schema_version, compatible, checksums[{file, ok}], counts{table: n}, excluded_tables[], absent_entity_types[], errors[]}` | validation only; import itself is CLI (D-6) |
| `GET /backups/drill` | admin | → `{last_import: ImportRun?, runbook: "docs/ops/restore-drill.md"}` | |
| `GET /system/health` | any | → `SystemHealth{backend_version, db_schema_version, broker_connected, ingest_lag_s, storage{bytes_used, pct}, last_restart_at, room_controllers[{device_id, last_seen_at, age_s}], sim_overlay: bool, mqtt_clients_seen}` | |
| `GET /safety-envelope` | any | → `{last_updated, entries[{key_path, value\|"TBD — needs domain expert input", unit?, source, comparator_used_by_rules[]}]}` | server parses `safety-rules.json` embedded at build (`//go:embed` of a CI-copied file); saffron/aquatic blocks are filtered out |
| `GET /facility/metadata` `PATCH /racks/{id}` `PATCH /tiers/{id}` | admin | `{position_ref?, notes?, doc_links?, tier_height_mm?, commissioning_ppfd?{value\|null}}` | metadata only (FR-FAC-05); PPFD write records `entered_by/at`, audited |

There is **no** `PUT/POST state/desired`, no schedule, no recipe, no `light.*`, `airflow.*`,
`irrigation.*`, `valve.*`, `co2.*`, `dosing.*` path anywhere in the spec; `spectest` asserts
the spec's only `POST /devices/{id}/commands` schema has `action: {const: "sys.identify"}`.

### 4.3 `FacilitySummary` (Home)

```yaml
FacilitySummary:
  as_of; verdict: {enum: [all_clear, issues, safety, unknown]}; verdict_reasons: string[]
  active_alert_counts: {safety, operational, advisory}
  safety_alerts: AlertRow[]                      # always all, first
  top_alerts: AlertRow[]  (≤ 10 non-SAFETY)
  device_counts: {online, offline, stale_telemetry, degraded, fault, unclaimed, quarantined}
  room_controllers: [{device_id, last_seen_at, age_s, stale: bool}]
  room_snapshot: {air_temp_c, air_rh_pct, vpd_kpa, co2_ppm, water_te02_c: ValueEnvelope|null}
  rack_counts: {total, active, idle, offline, with_alerts}
  platform: {ingest_lag_s, ingest_lag_threshold_s, broker_ok, last_backup: {finished_at, status}|null}
  sim: {virtual_devices, total_devices}
```

`verdict = all_clear` iff `active_alert_counts` all 0 ∧ `device_counts.offline+stale+degraded+fault = 0`
∧ every room controller `stale = false` ∧ `broker_ok` ∧ `ingest_lag_s ≤ 10` ∧ `last_backup.status = ok`
(or no scheduled run due yet, first 24 h). Any input unavailable → `unknown` with reasons.
`safety` dominates `issues`.

---

## 5. Backend Go interfaces

Only signatures and the state machines are given; bodies are the implementer's. All
functions take `context.Context` first; all repository calls take `store.Scope`.

### 5.1 Scope and repositories

```go
package store
type Scope struct{ OrgID uuid.UUID }              // constructed ONLY by authz.middleware (from session) or ingest (from topic <org> resolved via orgs.slug)
func ScopeFromSession(s *authz.Session) Scope
type Repos struct { Facility FacilityRepo; Users UserRepo; Sessions SessionRepo; Devices DeviceRepo; Profiles ProfileRepo;
                    Telemetry TelemetryRepo; LKS LastKnownStateRepo; Gaps GapRepo; Events EventRepo; Alerts AlertRepo; Rules RuleRepo;
                    Commands CommandRepo; Audit AuditRepo; Backup BackupRepo; Platform PlatformRepo }
// every method: func (r *x) Name(ctx context.Context, sc Scope, ...) (..., error)  — first param after ctx is Scope, enforced by §5.9
type DeviceRepo interface {
  Get(ctx, sc Scope, deviceID string) (Device, error)
  List(ctx, sc Scope, f DeviceFilter, pg Page) ([]Device, Cursor, error)
  UpsertUnclaimed(ctx, sc Scope, d NewUnclaimed) (Device, bool /*created*/, error)
  Transition(ctx, sc Scope, deviceID string, from, to Lifecycle, reason string, actor Actor) (Device, error)  // CAS on `from`
  SetCredential(ctx, sc Scope, deviceID string, c Credential) error
  ...
}
```

### 5.2 Ingest pipeline (`internal/ingest`)

```go
type Topic struct{ Org, Site, Room, Zone, Device, Leaf, Metric string }
func ParseTopic(t string) (Topic, error)                                  // ① strict regexp; error = counted drop
type Envelope struct{ V int; Msg string; Org, Site, Room, Zone, DeviceID, BootID string; Seq int64; TsSource time.Time; TMonoMs *int64; LinkSeq *int64; Clock string; Sim, Backfill bool }
func ValidateEnvelope(raw []byte, sch *schema.Set) (Envelope, json.RawMessage, ValidationOutcome) // ② outcome ∈ {ok, unsupported_version, invalid}; never panics
func CrossCheck(t Topic, e Envelope) (mismatch bool, reason string)       // ③
type Dedup interface { SeenOrMark(ctx, sc Scope, deviceID, bootID string, seq int64) (dup bool, err error) } // ⑤ ring (per device 4096 bitmap) + telemetry_dedup fallback
type GapDetector interface { Observe(ctx, sc Scope, deviceID, bootID string, seq int64, tsIngest time.Time, backfill bool) ([]GapEvent, error) } // ⑥ emits open/filled transitions
type Plausibility struct{ Ranges RangeSource; Units UnitSet; StuckN int }  // ⑦
func (p Plausibility) Gate(s Sample, dev DeviceCtx) Verdict                // Verdict{Quality, Reason, RangeSource, TimeBasis}; only downgrades
type RangeSource interface { Effective(class, metric string, dev DeviceCtx) (lo, hi float64, src string /*advert|class_fallback|none*/, ok bool) }
func ComputeVPD(t, rh Sample) (Derived, bool)                              // formula_version "vpd_air.tetens_fao56.v1"; returns false if not same (device, scope, ts_source)
type VPDPairer struct{ Window time.Duration /* 10 s */ }                   // holds unpaired T or RH; evicts with device_events{vpd_unpaired}
func ComputeDLI(ctx, sc Scope, tierID uuid.UUID, day civil.Date) (Derived, bool)   // nightly job per facility-local day; false when no PPFD (null, never zero)
type Staleness struct{ Floors map[string]int /* class → s */ }
func (s Staleness) MaxAge(class string, intervalS int) int                 // max(3*interval, floor)
func (s Staleness) Evaluate(now time.Time, lks LastKnownState) (stale bool, ageS float64)   // on ts_ingest
type Pipeline struct { ... }
func (p *Pipeline) Handle(ctx context.Context, m mqtt.Message) error       // ①..⑩ in HLD §4.1 order; every branch stores; only ParseTopic drops
```

`Pipeline.Handle` ordering is exactly HLD §4.1 ①–⑩ and is enforced by a table-driven test
that feeds each golden and asserts the row, the event and the LKS effect. Rules the
implementer must not re-derive: `backfill=true` or `quality=implausible-rejected` → return
after ⑧ (no LKS, no derived, no alerts); `clock=unsynced` or clock anomaly → `time_basis=ingest`;
`unit ∉ UnitSet` → `suspect{unit_unexpected}` **and skip range gate** (never convert);
`value` non-numeric for a `kind: number` metric → `implausible-rejected{not_a_number}`, `value_raw` kept;
RH 95–100 is `good`; negative VPD → derived row `implausible-rejected{rh_over_100}` and no LKS.

**Ingest outages (`outage.go`):** on `serve` start, insert `ingest_outages{kind: startup,
started_at: last platform_health.at or now-Δ, ended_at: now}`; on paho `ConnectionLost`
open a row, close on reconnect. `GapDetector` correlates a gap's `ts_from..ts_to` with these
rows (`cause=backend-down`), with `device_health_events` (`device-offline`), with
`platform_health.broker_connected=false` (`broker-down`), with `buffer_overflow`/`link_gap`
events (`unfillable`), else `unknown`. A gap becomes `filled` when backfilled rows cover
`seq_from..seq_to` (counted via `telemetry_dedup`).

### 5.3 Device lifecycle (`registry/lifecycle.go`)

```go
type Lifecycle string // unclaimed | claimed | commissioned | active | retired | quarantined
var transitions = map[Lifecycle]map[Lifecycle]Guard{
  unclaimed:    {claimed: guardClaim /* technician|admin; fingerprint last4 match; parent claimed+ */, quarantined: guardFingerprintMismatch},
  claimed:      {commissioned: guardAssign, retired: guardOperator, quarantined: guardFingerprintMismatch},
  commissioned: {active: guardActivate /* SUCCESS ack + seen=true on current assignment; failsafe_check ok */, claimed: guardProfileDiff, retired: guardOperator},
  active:       {claimed: guardProfileDiff /* keeps telemetry */, retired: guardOperator, quarantined: guardFingerprintMismatch},
  quarantined:  {}, // exit is R2 (revoke)
  retired:      {},
}
func (m *Machine) Transition(ctx, sc Scope, dev string, to Lifecycle, actor Actor, reason string) error // CAS in DB; audit row; illegal edge = ErrIllegalTransition (typed)
```

`sim` flip on a claimed device → `guardProfileDiff` path + `audit_log{category: security,
action: sim_flip}`. Command publishing checks `lifecycle ∈ {commissioned, active}` inside
the same transaction that allocates `cmd_seq`, so a retire racing an identify cannot
publish.

### 5.4 Health (`registry/health.go`) — two axes, one badge

```go
type Connectivity int  // Unknown, Online, Offline
type HealthInput struct{ Now time.Time; LastHeartbeat, LastTelemetry time.Time; LWT bool; HealthMsg *HealthPayload; ParentOffline bool; OfflineAfter time.Duration }
func StepConnectivity(cur Connectivity, in HealthInput) (next Connectivity, reason string, inherited bool)
func StepMetrics(now time.Time, lks []LastKnownState, s ingest.Staleness) (stale []string)
func Badge(conn Connectivity, fault bool, degraded string, stale []string) BadgeState
//   OFFLINE > FAULT > DEGRADED > STALE_TELEMETRY > ONLINE ; STALE_TELEMETRY only if conn == Online (AC-DEV-04b tests: offline device has stale=[]; parent offline ⇒ child offline inherited, no own alert)
```

Health ticks every 5 s over `device_health_current`; transitions write
`device_health_events` and call `alerts.Engine.OnHealth`. Zone `derived_offline` is
recomputed in the same tick (all serving devices OFFLINE or STALE).

### 5.5 Alert engine (`internal/alerts`) — persisted state machine

```go
type RuleEval struct{ Rule Rule; Override *Override; ScopeID string }
type Sample struct{ Value *float64; Bool *bool; Quality string; Reason string; TsIngest time.Time; Unclaimed bool; TestInjection bool }
type State string // idle | breaching | raised | clearing
func Admit(r Rule, s Sample) bool  // SAFETY/mirror: good OR suspect∈{stuck,clock_anomaly,cross_check}; never implausible-rejected, topic_mismatch, unit_unexpected, unclaimed
                                   // non-SAFETY: good OR suspect (any measurement reason); never rejected/unclaimed
func Breach(r Rule, ov *Override, s Sample) bool   // comparator table; hysteresis applies only in Clear()
func Clear(r Rule, ov *Override, s Sample) bool    // outside→ inside by hysteresis on the crossed bound: upper breach clears at hi−h, lower at lo+h
func Step(st EvalState, r RuleEval, s *Sample /*nil = tick without sample*/, now time.Time) (EvalState, []Effect)
```

Transitions (`Step`), identical for all severities unless marked:

| State | Input | Next | Effect |
|---|---|---|---|
| idle | admitted breach sample | breaching{since=now} (SAFETY: **raised** immediately, duration 0) | SAFETY: Raise |
| breaching | breach continues, now−since ≥ duration_s | raised | Raise (once) |
| breaching | non-breach admitted sample | idle | — |
| breaching | tick, no sample | breaching | — (silence never raises) |
| raised | clear sample (per `Clear`) | clearing{since=now} | — |
| raised | breach sample | raised | update `current_snapshot` |
| raised | tick, no admitted sample for > `staleness max_age` | raised, `monitor_silent_since=t` | SAFETY: alert row `monitor_silent_since` set; **never resolves** |
| clearing | clear continues, now−since ≥ clear_s | idle | AutoResolve{reason: "auto: condition cleared"} |
| clearing | breach sample | raised | — |
| clearing | tick, no sample | SAFETY: **raised** (silence is not clearance) / non-SAFETY: clearing | — |
| any | rejected/untrusted sample | unchanged | non-SAFETY device-health path handles it |

Named tests (`engine_test.go`): `TestSafetyNeverAutoResolvesOnSilence`,
`TestSafetyResolveRefusedWhileMonitorSilent`, `TestSafetyRaisesOnSuspectStuck`,
`TestSafetyIgnoresTopicMismatch`, `TestSafetyIgnoresImplausibleRejected`,
`TestSafetyIgnoresUnclaimed`, `TestBandRuleDurationDebounce`, `TestRHHysteresisClearsAt65`,
`TestRestartMidConditionDoesNotReRaise` (state loaded from `alert_eval_state`),
`TestOfflineCollapsesStaleChildren`, `TestRootCauseSuppressedUnderParent`,
`TestBackfillNeverEvaluated`, `TestTestInjectionFlagPropagates`.
Operator resolve: allowed iff state ∈ {clearing, idle} for the rule×scope; SAFETY also
requires `last_sample_ts ≥ now − clear_s` and admitted. Ack: SAFETY requires
`action_code ∈ rule.action_codes`. Renotify: `renotify_s` (null = never) per rule, only
while unacknowledged. Notify: `console` = row exists; `email` via SMTP for configured
channels; SAFETY channels are the union of seeded console + any added, never fewer.
`noisy.go` recomputes the 30-day stats hourly.

### 5.6 Command dispatcher (`ingest/publisher.go`, `sysaction/allowlist.go`)

```go
package sysaction
const Capability = "control.compute"
var AllowList = [...]string{"sys.identify"}      // compile-time; AC-DEV-03e test asserts len==1 && [0]=="sys.identify" and that Publisher.Publish is the only caller of mqtt.Client.Publish in backend/ (go/analysis check)
package ingest
type Publisher struct{ ... }
func (p *Publisher) Identify(ctx, sc Scope, deviceID string, durationS int, actor Actor) (Command, error)
// allocates cmd_id = ulid.Make(), cmd_seq = command_seq.next++, expires_at = issued_at + 60 s; QoS 1; retains nothing
// refuses (typed errors → API codes) unless: lifecycle ∈ {commissioned, active}; capability_index has control.compute with actions ∋ sys.identify; durationS ≤ params.max_duration_s; no command in flight (busy)
func (p *Publisher) OnAck(ctx, sc Scope, a AckPayload) error     // matches cmd_id; sets acked; latency; DEGRADED{command_timeout} on TIMEOUT via registry
func (p *Publisher) Sweep(ctx, now time.Time)                    // issued && now > issued_at + 30 s && no ack → unacknowledged (backend state only; never publishes an ack)
```

`sys.ping` has **no Phase 1 caller** in the backend; it stays in the schema enum, the edge
router and the firmware, and is exercised by the harness principal only (brief §10.1 C(c)
resolved by dropping it from the backend allow-list).

### 5.7 Backup and restore (`internal/backup`)

Bundle `trophic-export-<org>-<UTC>.tar.gz.age`: age (X25519 recipient derived from the
`backup_key` identity secret; key never in the bundle; `age -d -i backup_key` works without
our binary). Inside: `manifest.json`, `manifest.sha256`, `entities/<table>.jsonl` for every
table in §3.2–3.8 **except** `sessions, login_attempts, totp_enrolments, telemetry_dedup,
alert_eval_state? (included — it is state), platform_health (excluded, ephemeral)`;
`audit/audit_log.jsonl`; `telemetry/rollups_{1m,1h,1d}.jsonl`; `telemetry/raw_<from>_<to>.jsonl`
(default 30 d); `telemetry/derived_<from>_<to>.jsonl`. `manifest` = `{bundle_schema_version: 1,
app_version, db_schema_version, exported_at, org, counts{table: n}, files[{path, sha256, rows}],
raw_window_days, excluded_tables[...], absent_entity_types: [recipes, batches, inventory, orders],
requires_secrets: [totp_key, backup_key]}`. `totp_secrets` rows are exported **as stored**
(ciphertext) with `key_version`; restore therefore needs the same `totp_key` — the runbook
carries it (F-3). Scheduler: cron `0 2 * * *` facility tz, retention 14, writes `backup_runs`,
failure → `platform.backup_failed`.

```go
func Export(ctx, sc Scope, opts ExportOpts, w io.Writer) (Manifest, error)
func Validate(ctx, r io.Reader, key age.Identity, dbSchema int) (Report, error)   // every sha256; schema compat = manifest.db_schema_version ≤ current; tampered byte → error naming the file
func Restore(ctx, r io.Reader, key age.Identity, policy Policy /* ImportAsNew | Replace */, db *pgxpool.Pool) (Report, error)
//   ImportAsNew: fails if orgs.slug exists; Replace: truncates org-scoped rows for that org in a single tx after `--confirm <org-slug>`
//   after commit: registry.MaterialiseACL(); audit row {actor: system, action: restore}; ingest_outages row covering exported_at..now; no user is created, ever
```

### 5.8 Authz middleware and ACL materialiser

```go
func RequireSession(next http.Handler) http.Handler   // cookie → sessions (sha256 lookup); idle 12 h / absolute 7 d; touches last_seen_at ≤ 1/min; injects Scope + Principal
func RequireRole(roles ...Role) func(http.Handler) http.Handler
var Matrix = map[Action][]Role{ ... HLD §7.1 table verbatim ... }   // handlers call authz.Can(p, action) and tests assert Matrix == the OpenAPI x-roles extension per operation
func Login(ctx, email, password, totp string, ip net.IP) (*Session, error)   // argon2id verify always runs (constant-time dummy hash on unknown email); totp step replay guard; one generic error
func MaterialiseACL(ctx, sc Scope, repos, out io.Writer, extra io.Reader /*nil in prod*/) error
//   generates §2.1 lines for backend-ingest + every device with credential_type=mtls-ecc-p256 and acl_prefix; extra appended only when TROPHIC_SIM_OVERLAY=1; writes atomically (tmp+rename) + aclfile.sha256
```

Mosquitto reload: `ops/mosquitto/entrypoint.sh` runs mosquitto in the background and every
5 s compares `aclfile.sha256`; on change sends `SIGHUP` to mosquitto. The backend runs
`MaterialiseACL` on every `serve` start and after every credential change/restore. No docker
socket, no dynsec plugin.

### 5.9 Org-scoping coverage gate (`store/repotest`)

1. Every repository is an interface in `store`; `repotest.TestScopeSignature` walks each
   interface with `reflect` and fails if any method's second parameter is not `store.Scope`.
2. `repotest.Registry` maps `"<Repo>.<Method>"` → a `func(t, orgA, orgB Scope)` that seeds
   rows for both orgs and asserts the call under `orgA` returns/mutates zero `orgB` rows
   (`SELECT count(*) … WHERE org_id = B` before/after).
3. `TestScopeCoverage` fails the build if any interface method is missing from `Registry`.
   Adding a repository method without its scoping test is a compile-and-test failure, not a
   review comment.
4. `spectest` additionally asserts no OpenAPI parameter or property is named `org_id`/`org`.

---

## 6. Virtual room controller (`sim/`)

### 6.1 Interfaces

```go
type Clock interface { Now() time.Time; NewTicker(d time.Duration) Ticker; Sleep(ctx, d) }        // real.go / fake.go (scenario time-warp for CI short-forms)
type BufferStore interface { Append(m Outbound) (seq int64, err error); Iter(fromSeq int64, fn func(Outbound) bool) error; Trim(keepFrom int64) error; Persist() error; Load() (lastSeq int64, bootID string, err error) }
type RackLink interface { Frames(ctx) <-chan Frame; Send(ctx, f Frame) error; Attach(deviceID string) error; Detach(deviceID string) error; Faults() FaultSink }
type Broker interface { Connect(ctx, opts ConnectOpts /*LWT topic+payload, CleanSession=false*/) error; Publish(ctx, topic string, retained bool, payload []byte) error; Subscribe(ctx, filter string, h Handler) error; Disconnect(graceful bool) error; Events() <-chan BrokerEvent }
```

`inproc.go`: Go channels per virtual device (`vdevice` goroutines write frames); `serial.go`:
`go.bug.st/serial` JSONL with the 4 KB byte guard. `file.go` BufferStore: append-only
segment files under `--state-dir`, capacity **27 000 messages** (6 h at nominal), overflow →
drop oldest + `event{buffer_overflow, seq_from, seq_to}`; `seq` and `boot_id` are persisted
so a restart continues the sequence under a new `boot_id` (never reuses numbers).

### 6.2 Controller loop (`vroom/controller.go`)

1. Load `BufferStore` → `boot_id = new`, `seq = 1`. Publish `event{boot}` for `rmc-01`.
2. Connect with LWT on `…/room/rmc-01/state/health` `{status: offline, reason: lwt, seq: 0}`, retained.
3. On connect: publish own `capability` (retained) and `health{online}`; for every attached
   node publish its retained `capability` and `health{online}`; then **replay** every ring
   entry with `seq > last_acked_publish` with `backfill: true` in `seq` order (paho QoS 1
   completion = acked).
4. Bridge: each `Frame` → stamp envelope (`ts_source = Clock.Now()`, `clock = ntp` unless the
   `clock-frozen` fault is on → `unsynced`… see §6.5), `seq++`, append to ring, publish. Publish
   failure or disconnected → ring only (replayed at 3).
5. Health-on-behalf: per node `link_timeout_s = 10`: no frame → publish retained
   `health{offline, bus_timeout}` for the node; frame resumes → `health{online}` and, if
   `link_seq` jumped, `event{link_gap}`.
6. Heartbeat 30 s own `health` with `buffer{depth, capacity: 27000, dropped}`.
7. Command router (§6.3). Subscribe `…/<room>/+/+/cmd`.

### 6.3 Command router (`vroom/cmdrouter.go`)

Order: parse+schema → `device known && attached` else drop+log → `lifecycle` known from
the device's own claim flag (`vdevice.Claimed`, set by the harness when the backend claims —
the sim mirrors registry state by watching its own `event{claim}` ack… **decision:** the
virtual device treats itself as claimed once it has received any `cmd` after the backend's
claim call is observed through `simctl claim <id>` in scenarios; for CI the scenario step
`claim` follows the API call) → `device_unclaimed` → `expires_at < now` → `expired` → dedup
ring 32 per device: hit → replay stored ack `duplicate: true` → `cmd_seq < last` →
`stale_sequence` → validate against the device's **own** advert: class ∉ capabilities →
`capability_not_advertised`; class ∈ capabilities ∧ (class,action) ∉ commands or `sys.*` ∉
enum → `action_not_supported`; params (`duration_s > max_duration_s`) → `params_out_of_range`;
in-flight on identify → `busy` → forward over RackLink; wait `ack_timeout_s = 5`, retry same
`cmd_id` ≤ 2; then `ack{TIMEOUT}` + node `health{degraded, command_timeout}`; node offline →
park until `expires_at` → `ack{EXPIRED}` + `event`.

### 6.4 Virtual devices and profiles as data

`vdevice.New(profile Profile, params SimParams, clk Clock) *Device` reads `tiers`,
`capabilities[].metrics`, `interval_s`, `on_change`; it emits exactly the advertised metrics
(a profile without `airflow.tacho` emits no tacho — tier count and channel set are never
constants). Profiles (§1.4) as HLD §3.3 with these tag decisions: rack T/RH node `tag: null`
(no tag in contracts); `LD-NN` per rack; `LSH-04` on every rack with `instance = rack index`
and `tag_note: "contracts tag is per-rack; uniqueness by (device_id, class)"`; `io-terrace`:
`TE-01`, `LT-02` tagged, `sense.ph`/`sense.ec` **`tag: null`, `vessel: "TK-01"`** (brief §10.1
C(b): no analyser tag exists for the terrace block); `io-skid`: `TE-02`, `AT-03` (EC, mS/cm),
`AT-04` (pH); `io-room`: `sense.temp_rh` supply + return **only** — E-stop/floor-flood
omitted ([unknown]). `esp32-skeleton-v0.json` is captured by `virtual-room --capture
<serial>` and CI diffs it against the firmware host test's emitted advert.

### 6.5 Diurnal generator (`vdevice/diurnal.go`) — all values `sim_param`, unsourced

```yaml
sim_params (defaults; every key is echoed into the scenario report as unsourced):
  photoperiod: {on_h: 18, off_h: 6, stagger_min_per_rack: 3, lights_on_local: "06:00"}
  rack_air_temp_c: {day_target: 23.0, night_target: 21.0, tau_min: 40, noise_sigma: 0.15}
  rack_air_rh_pct: {day_target: 63, night_target: 68, night_excursion: {peak: 72, minutes: 30, every_h: 24, start_offset_h: 2}, noise_sigma: 1.0}
  room_supply: {temp_offset: -0.6, rh_offset: +1}; room_return: {temp_offset: +0.4, rh_offset: -1}   # shared slow component
  fan_tacho_hz: {plateau: 120.0, noise_sigma: 0.5, day_night_delta: 0}    # constant day and night; a drop at lights-off is a sim bug
  co2_enrichment: {model: enrichment-on, day_band: [1100, 1400], night_floor: 600 /*ambient floor unsourced*/, noise_sigma: 25}
  co2_safety_mirror: {offset: +60, noise_sigma: 25, clamp_max: 4800}  # never ≥ 5000 without override
  te01_c: {mean: 16.0, swing: 4.0, noise: 0.1}; te02_c: {track: rack_air_temp, lag_min: 90, afternoon_push: {to: 26.5, minutes: 40, every_h: 24}}  # crosses 26 once/day, never 28
  ph: {mean: 5.9, drift_per_day: 0.15, excursion: {to: 6.3, minutes: 20, every_h: 36}}
  ec_ms_cm: {mean: 1.6, noise: 0.03, label: unsourced-sim-placeholder}
  lt02_pct: {mean: 91, blowdown_step_at_local: "02:00", step: -3}
  light_state_report: from photoperiod (declared, not actuated)
```

Consequence the LLD verifies (brief §5.1 RH row): the night RH excursion to 72 % raises
`climate.rh_band` after 300 s; with hysteresis 5 the clear condition is `RH ≤ 65` for 300 s,
which the day target (63 ± 1) satisfies within ~2 τ after lights-on, so **the ADVISORY
resolves each day** — the soak asserts "each RH ADVISORY raised at night is auto-resolved
before the next lights-off". If it does not, the hysteresis borrowed from control is the
cause and it is a finding, not a threshold change.

### 6.6 Fault switches (`internal/faults`) — `simctl`/scenario only

`offline{device|edge-ungraceful|edge-graceful}`, `stale{metric|all|clock-frozen}`,
`implausible{range|stuck|non-numeric|unit-mismatch}` exactly as IoT §6 (oracles in §10);
`backend-only` is a compose action in the scenario runner (`docker stop backend`), not a
sim switch. **`override{device, metric, value, minutes}`** (test-only): sets the emitted
value; the device marks the sample `sim_test_injection: true` in `telemetry.detail` — a
**sim-side field outside the schema's enumerated properties** that the backend's ingest
copies to `alerts.is_test_injection` when it raises. The `72h-soak.yaml` loader **refuses**
any `override` step (`scenario: soak` class); only `safety-override.yaml` may use it.

### 6.7 `simctl` control surface and scenario format

`simctl` talks HTTP over a **unix socket** (`/run/simctl.sock` inside the container) or,
in the overlay only, `virtual-room:7777` on the overlay-internal network; never published
on a host port; no auth (unreachable outside the overlay, F-9). Endpoints: `POST /faults`,
`DELETE /faults/{id}`, `GET /state`, `POST /pki/init`, `POST /capture`, `POST /scenario/run`.

```yaml
# sim/scenarios/72h-soak.yaml
schema: sim-scenario/1
class: soak                      # soak | e2e ; soak forbids override
duration: 72h
time_scale: 1                    # CI short-form uses 60 with FakeClock
fixture: {racks: 11, rack_profile: rack-controller-std-v0, room: room-controller-v0, io: [io-terrace-v0, io-skid-v0, io-room-v0], interval_s: 30}
sim_params: {}                   # overrides of §6.5 defaults
timeline:
  - at: 6h   action: infra.backend_stop   for: 20m
  - at: 18h  action: fault.offline  variant: edge-ungraceful  for: 30m
  - at: 30h  action: fault.stale    variant: metric target: {device: rc-03, metric: air_temp_c} for: 1h
  - at: 42h  action: fault.implausible variant: range target: {device: rc-07, metric: air_rh_pct, value: 118} for: 5m
  - at: 54h  action: fault.offline  variant: device target: {device: rc-11} for: 45m
assert:
  - gaps.open == 0
  - gaps.unfillable.all_have_reason
  - aggregates.complete: [1m, 1h]
  - ingest_lag.non_monotonic
  - alerts.safety.count == 0
  - alerts.rh_band.each_resolves_before_next_night
```

*(continued in `06-lld-part-2.md`)*
