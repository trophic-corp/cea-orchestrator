# 06 — Low-level design: phase-1-foundation (part 2 of 2)

Continues `06-lld.md` (§1–§6). Same author, date, inputs and scope guard. Sections here: §7
ESP32 skeleton, §8 frontend, §9 compose and CI, §10 test plan, §11 defaults and unknowns,
§12 security-review disposition, §13 flags.

---

## 7. ESP32 skeleton (`firmware/rack-controller/`)

### 7.1 Relay hard gate (F-8, brief §5.4) — build-time and README

**Build-time assertion.** `components/board/CMakeLists.txt` runs a check that fails the
build if any of the following is true: a Kconfig symbol matching `CONFIG_TROPHIC_RELAY_*`
exists; any source under `firmware/rack-controller/` (excluding `README.md`) contains the
tokens `relay`, `solenoid`, `fill_gpio`, `drain_gpio`, `RELAY_CH` (case-insensitive grep,
run as a CMake `execute_process` and duplicated as a CI step); any call to
`gpio_set_direction(..., GPIO_MODE_OUTPUT*)` or `gpio_config` with an output mode appears
outside `components/sysaction/led.c`. The single permitted output is the status LED.

**README gate (verbatim text the firmware-engineer must place in `README.md` and the
`sdkconfig.defaults` header):**

> No Phase 1 bench or HIL rig powers a rack-controller board with a relay module, solenoid,
> fan driver or LED driver connected; a Phase 1 bench is a 3.3 V/5 V/24 V logic rig with no
> loads. No firmware version that maps a relay pin is flashed onto a board with relays fitted
> until the relay drive-polarity / boot-state / brown-out bench check for that board revision
> has passed and is recorded (contracts `safety/valve-fail-states.md`, "Consequence for
> procurement and commissioning"). This image (v0) maps no relay pin and cannot pass that
> gate by construction.

**Status LED pin (F-8 (ii)).** `CONFIG_TROPHIC_STATUS_LED_GPIO` is set in
`board_<variant>.h` with a `_Static_assert` against the variant's strapping set
(`board_esp32.h`: `{0, 2, 5, 12, 15}`; other variants define their own set) and against
every pin listed in `BOARD_RESERVED_ACTUATOR_PINS[]` — a table of pins the board
descriptor *reserves for* relay/PWM/0–10 V lines **without mapping them** (names only,
no direction, never configured). The board header records `// why: <pin> is not a strapping
pin and not on the relay header` next to the choice. Variant is `[unknown]` (H2): the
default board is `esp32-devkitc-generic` and `TROPHIC_BOARD` must be set explicitly.

### 7.2 Boot sequence (`main/boot.c`)

1. `esp_reset_reason()` recorded; **no GPIO is touched** except: inputs for leak/LSH-04
   contacts (pull-up, `GPIO_MODE_INPUT`), PCNT inputs for 4 tachos, I2C SDA/SCL, UART
   TX/RX, status LED (`GPIO_MODE_OUTPUT`, driven to off).
2. NVS init (boot counter + `link_boot_id` only; no credentials exist in this image).
3. I2C probe at `CONFIG_TROPHIC_I2C_ADDR_SHT` (0x44/0x45): found → `sense.temp_rh`
   advertised with `range` from the driver's `trh_driver_t.range()` (or omitted if the driver
   cannot state it → backend class fallback); not found → not advertised,
   `health{degraded, sensor_init_failed}` if the board descriptor says it should exist.
4. Contacts: advertised iff `CONFIG_TROPHIC_CONTACT_LEAK_GPIO` / `_LSH04_GPIO` are set
   (`kind: switch`, `on_change: true`, debounce `CONFIG_TROPHIC_CONTACT_DEBOUNCE_MS`
   default 50 — an LLD choice, not a safety timing).
5. Tachos: advertised iff PCNT units configured; metric `fan_tacho_hz`, unit `Hz`.
6. Board descriptor (`board.c`) contributes actuator **counts** only: `valve.control ×4
   {failsafe_state: CLOSED, duty: fill}`, `×4 {OPEN, drain}`, `airflow.speed ×4`,
   `light.dim ×4`, `irrigation.flood ×4`, `light.state_report ×0` (no LED driver, OQ-14) —
   **empty by default** (`BOARD_DESCRIBE_ACTUATORS 0`) on a bare bench board.
7. `commands = [{control.compute, [sys.identify, sys.ping]}]`; `control.compute.params =
   {role: rack, max_duration_s: 60}`; optional `sys.heap_free` metric (`bytes`, 60 s) when
   `CONFIG_TROPHIC_DIAG_HEAP=y`.
8. UART start; emit `event{boot}` frame, then `capability` frame (must be ≤ 4096 B —
   `advert_build()` returns an error and the build's host test fails if the golden advert
   exceeds it), then `health{online}` and the 30 s timers.

### 7.3 Sample path (register image)

`sample_path` owns `struct reg_image { channel[N] { value, unit, quality, changed, seq }, seq }`;
a 30 s timer snapshots it into one `telemetry` frame **per channel** (one metric per
frame, matching the MQTT one-sample-per-message rule) with `link_seq++` per frame and
`t_mono_ms = esp_timer_get_time()/1000`. Contacts additionally emit on change. No timestamp
is generated on the ESP32 (no clock). R2 swaps the serialiser for a Modbus register read
without touching `reg_image`.

### 7.4 `rack-link/0` UART framing

`CONFIG_TROPHIC_UART_NUM/BAUD` (default UART0 for USB-serial; 115200; RS485 DE/RE pin only
when `CONFIG_TROPHIC_RS485=y`). Writer: cJSON compact, `\n` appended, byte length checked
≤ 4096 before write. Reader: line accumulator with a 4096-byte cap (overflow → discard to
next `\n`, count `link.errors`), parse, dispatch `cmd` frames only.

### 7.5 `sys.identify` and `sys.ping`

`sysaction/identify.c`: on `cmd{sys.identify, duration_s}`: `duration_s > 60` →
`ack{REJECTED, params_out_of_range}`; else start a 2 Hz LED blink task bounded by
`min(duration_s, 60)` s, re-issue extends to the new deadline (never stacks), reply
`ack{SUCCESS, state_after: {identify: {active: true, until_mono_ms}}}`; `health` frames carry
`identify.active` while blinking. `sys.ping`: `ack{SUCCESS, state_after: {ping: {uptime_s,
fw_version, clock: unsynced, seq}}}`. Any other class/action: `capability_not_advertised`
or `action_not_supported` by the same two-list check as the sim (§6.3). Dedup ring of 32
`cmd_id` per board with stored acks. The image contains **no** `esp_wifi`, `esp-tls`,
`mqtt`, or `sntp` component: `CMakeLists.txt` sets `EXCLUDE_COMPONENTS "esp_wifi mqtt
esp-tls lwip"` where the IDF version allows, and the host test asserts the built ELF has no
`esp_wifi_init` / `esp_mqtt_client_init` / `sntp_init` symbols.

### 7.6 Host conformance test (`test/host/`)

Builds `advert`, `racklink`, `sample_path` for the host with cJSON; emits the advert for the
`esp32-skeleton-v0` board config and compares to `sim/profiles/esp32-skeleton-v0.json` after
canonicalisation (fingerprint field masked); parses `golden/racklink/cmd.jsonl` and emits
the expected acks. Run in the `firmware` CI job before `idf.py build`.

### 7.7 `firmware/room-controller/README.md`

> Empty until OQ-3 closes as ADR-0007 (`docs/pipeline/oq3-room-controller-platform/00-decision-brief.md`).
> The virtual room controller in `sim/cmd/virtual-room` is a simulator, not this. ADR-0007
> must reference `docs/iot/schema/v1/rack-link-0.md` and its sunset.

---

## 8. Frontend (`frontend/`)

### 8.1 Route table (Vue Router, history mode; all except `/login` behind `requireSession`)

| Route | View | API calls (vue-query keys) | Poll (s) |
|---|---|---|---|
| `/login` | `Login.vue` | `POST /auth/login`, `GET /public/safety-banner` | 30 (banner) |
| `/` | `Home.vue` | `GET /facility/summary` | 5 |
| `/grow` → redirect to the single room | — | `GET /facility/tree` | — |
| `/grow/rooms/:id` | `RoomView.vue` | `/facility/tree`, `/rooms/{id}/snapshot` | 10 |
| `/grow/racks/:id` | `RackView.vue` | `/racks/{id}/snapshot` | 10 |
| `/grow/tiers/:zoneId` | `TierView.vue` + `TelemetryChart.vue` | `/zones/{id}/snapshot`, `/zones/{id}/telemetry`, `/zones/{id}/events` | 10 / 30 / 30 |
| `/devices` | `FleetList.vue` | `/devices?health=&claim_state=` | 10 |
| `/devices/commission` | `commission/Wizard.vue` | see §8.5 | 5 (steps 0, 4) |
| `/devices/:id` | `DeviceDetail.vue` | `/devices/{id}`, `/capabilities`, `/commands`, `/events` | 10 |
| `/alerts` (`?tab=active\|history\|rules`) | `AlertCenter.vue` | `/alerts?state=`, `/alert-rules` | 5 / 30 / 60 |
| `/alerts/:id` | `AlertDetail.vue` + `AckForm.vue` | `/alerts/{id}`, `POST ack`, `POST resolve` | 5 |
| `/settings` (`users\|backup\|system\|notifications\|facility\|safety-envelope`) | `Settings.vue` sections | `/users`, `/backups`, `/system/health`, `/notification-channels`, `/facility/metadata`, `/safety-envelope` | 30 |

Polling intervals and backoff are **[proposed default, unsourced]**: on fetch failure the
interval doubles to a 60 s cap and `connection.ts` sets `backendReachable=false`; success
resets. The shell uses the summary query's `as_of` for the strip.

### 8.2 App shell (`shell/`)

`AppShell.vue` = `SafetyBanner` (rendered when `summary.safety_alerts.length > 0`;
no close control; text includes "Hardware interlock state — the console mirrors this; it
does not make the room safe" when `is_interlock_mirror`; tap → `/alerts/:id`) →
`ConnectionStrip` (three facts: browser↔backend from the query state; backend↔broker from
`platform.broker_ok`; room controller from `room_controllers[].age_s/stale`) → `<RouterView>`
→ `NavTabs` (bottom bar ≤ 768 px, left rail above; badge = `active_alert_counts` with SAFETY
first). `SimChip` renders "SIMULATED: n of m devices are virtual" when `summary.sim.virtual_devices > 0`.
Disconnected: all `ValueEnvelope` instances get `dimmed`, ages keep ticking from the last
`as_of`, Home verdict forced to `unknown` client-side (never `all_clear`).

### 8.3 `ValueEnvelope.vue` — the only number renderer

Props: `v: ValueEnvelope`, `label?`, `compact?`. Renders `value unit (age)`; age formatted
per UX §4 rule A2 from `age_s` + local tick since `as_of`; `stale` → text tag `STALE` and
dim; `quality=suspect` → `?` tag with reason on tap; `quality=implausible_rejected` never
reaches a headline (the API returns the last admitted sample; the component still handles
it by rendering `—` + `REJECTED`); `provenance` → `ProvenanceTag.vue` text tags:
`calculated` → `≈` prefix + `CALC` (+ formula_version and inputs with their ages on tap),
`estimated` → `EST`, `operator_entered` → `ENTERED by <entered_by>` on tap, `sim` → `SIM`;
`instrument_tag` on tap; `is_test_injection` → `TEST` tag. `value=null` → `—` with
"no admitted sample". An ESLint rule (`no-restricted-syntax` on
`{{ … .value }}` outside `ValueEnvelope.vue`) plus a vitest snapshot walk over every view with
a fixture summary assert no bare number renders (UX AC 1).

### 8.4 Alert center rules (client-side mirrors of §5.5)

No component contains the word "dismiss"; no snooze. `AckForm.vue`: for SAFETY the submit
button is disabled until an `action_code` is chosen (`other` requires a note); the server's
412 is rendered verbatim. Resolve button disabled with `resolve_blocked_reason` text when
`resolvable=false`; a 412 on submit re-fetches and shows the reason. Active tab groups one
row per `rule × scope`; children with `suppressed_under` are not rows — they appear in the
parent's detail as "also affects". Rules tab is read-only (D-7); SAFETY rows show
`source_citation` and `protects_text`; `editable=false` rows have no edit affordance at all.

### 8.5 Commissioning wizard (`commission/`)

Step 0 polls `GET /devices?claim_state=unclaimed`. Step 1: select → type last-4 → `POST claim`
(412 → "fingerprint mismatch; device quarantined" and stop). Step 2: `GET capabilities`
renders `CapabilityList` grouped by scope with `expected_counts`, `count_mismatches`,
`failsafe_check` per valve (mismatch = red "FAULT — advertised X, `safety-rules.json`
`valve_and_actuator_fail_safe_states.<key>` says Y") and `unrecognised[]` as "recorded, not
rendered"; the tick "profile matches the hardware I expect" enables `POST verify`; a
blocking fault disables it. Step 3: rack picker from `/facility/tree`, instance map defaults
tier 1..`tiers`; conflict 409 shows the existing controller and a link to retire it. Step 4:
`POST commands {sys.identify, duration_s: 10}` → poll `GET /commands/{id}` until `backend_state
≠ issued` or 30 s; render issued → ack {result, reason, latency_ms} and the device's reported
`identify.active`; then "Did the controller on Rack N blink?" → `POST confirm-identify`
(`confirmed_by_name` free text if not the logged-in user); No/REJECTED/TIMEOUT stays on 4
with "retry" (new cmd). Step 5: summary of `commissioning_steps[]` → `POST activate`. The
wizard is resumable from `DeviceDetail.commissioning_step`. Forms use **FormKit open-source
core** (no Pro → no key; if Pro is ever adopted the key is a CI secret, never in `frontend/`).

### 8.6 Settings sections and security carry-forwards

Users (admin): list, add (shows the one-time enrolment URL once), role/disable, TOTP reset.
Notifications: per-severity channel list; SAFETY rows have no delete. Backup & export:
schedule (read-only), last run, `Export now` → job poll → download; recent bundles;
`Upload bundle → validate` shows `ValidationReport`; import itself says "run
`backend restore` per `docs/ops/restore-drill.md`" (D-6). System: `/system/health`.
Facility: metadata edit (rack position/notes/doc links, tier height, `commissioning_ppfd`
with entered-by). Safety envelope: table from `/safety-envelope`, TBD entries render the
literal string, never a number.

Carry-forwards (ADR-0009 review / F-13): `package-lock.json` committed; CI runs `npm ci`
only; `.npmrc` `ignore-scripts=true` (build scripts of allow-listed packages — `esbuild`,
`vue-demi` — are run explicitly via `npm rebuild <pkg>` in `postinstall:allowlist`); `npm audit
--audit-level=high` as a CI step; dependency bumps by reviewed PR only (Dependabot config,
no auto-merge); `vite-plugin-pwa` transitive advisory (`serialize-javascript`) re-checked at
scaffold and recorded in the PR; PWA `workbox`: `globPatterns: ['**/*.{js,css,html,svg,woff2}']`,
`navigateFallback: '/index.html'`, `runtimeCaching: [{urlPattern: /\/api\//, handler:
'NetworkOnly'}]`, `cleanupOutdatedCaches: true`; ESLint `no-restricted-imports` for
`../backend`, `../sim`, `../firmware`; no MQTT client dependency permitted (CI greps
`package-lock.json` for `mqtt` and fails). CSP set by the backend: `default-src 'self';
connect-src 'self'; frame-ancestors 'none'; img-src 'self' data:`; HSTS on 8443.

---

## 9. Compose and CI

### 9.1 `docker-compose.prod.yml` (uncomment/replace)

```yaml
services:
  backend:
    build: {context: ., dockerfile: backend/Dockerfile}
    restart: unless-stopped
    ports: ["8443:8443"]
    environment: {TROPHIC_CONSOLE_DIR: /srv/console, TROPHIC_ACL_OUT: /mosquitto/acl/aclfile, TROPHIC_BROKER_URL: ssl://mosquitto:8883, TROPHIC_DB_HOST: timescaledb}
    secrets: [db_password, db_migrate_password, broker_ca_cert, backend_client_cert, backend_client_key, backend_tls_cert, backend_tls_key, totp_key, backup_key, smtp_password]
    volumes: [backups:/var/lib/trophic/backups, mosquitto-acl:/mosquitto/acl]
    depends_on: {mosquitto: {condition: service_healthy}, timescaledb: {condition: service_healthy}}
    healthcheck: {test: ["CMD", "/trophic", "healthz"], interval: 15s}
  # frontend: intentionally absent — the console is a static bundle served by backend (ADR-0009)
  mosquitto:
    build: ./ops/mosquitto                 # FROM eclipse-mosquitto:2.0.20@sha256:<pinned at PR time; CI asserts '@sha256:' present>
    restart: unless-stopped
    ports: ["8883:8883"]                   # the ONLY listener
    volumes: [mosquitto-data:/mosquitto/data, mosquitto-acl:/mosquitto/acl:ro]
    secrets: [broker_ca_cert, broker_server_cert, broker_server_key, broker_crl]
    healthcheck: {test: ["CMD-SHELL", "mosquitto_sub -h localhost -p 8883 --cafile /run/secrets/broker_ca_cert --cert … -t '$$SYS/#' -C 1 -W 3 || exit 1"]}
  timescaledb:
    image: timescale/timescaledb:2.17.2-pg16@sha256:<pinned>
    restart: unless-stopped
    environment: {POSTGRES_PASSWORD_FILE: /run/secrets/db_password, POSTGRES_DB: trophic}
    volumes: [timescale-data:/var/lib/postgresql/data, ./ops/db/init:/docker-entrypoint-initdb.d:ro]   # creates trophic_migrate/trophic_app roles
    secrets: [db_password, db_migrate_password]
    # no host port
volumes: {mosquitto-data: {}, mosquitto-acl: {}, timescale-data: {}, backups: {}}
secrets: {db_password: {file: ./.secrets/db_password}, db_migrate_password: {file: …}, broker_ca_cert: …, broker_server_cert: …, broker_server_key: …, broker_crl: …,
          backend_client_cert: …, backend_client_key: …, backend_tls_cert: …, backend_tls_key: …, totp_key: …, backup_key: …, smtp_password: …}
```

`ops/mosquitto/mosquitto.conf`:

```
listener 8883
protocol mqtt
allow_anonymous false
require_certificate true
use_identity_as_username true
cafile  /run/secrets/broker_ca_cert
certfile /run/secrets/broker_server_cert
keyfile  /run/secrets/broker_server_key
crlfile  /run/secrets/broker_crl
tls_version tlsv1.2                # minimum; TLS 1.3 negotiated when offered
ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305
acl_file /mosquitto/acl/aclfile
persistence true
persistence_location /mosquitto/data/
max_queued_messages 60000          # ≥ 2 × 27 000 ring
max_inflight_messages 100
message_size_limit 65536
```

Broker server certificate is issued by the **same facility CA** (`ops/pki/issue-service.sh
broker`), SAN = the compose service name and the host's LAN name; every client (`backend`,
`sim`) verifies against `broker_ca_cert` with hostname check and **no insecure-skip-verify
flag exists in either codebase** (CI greps `InsecureSkipVerify` → fail). Certificate validity:
devices 2 years, services 1 year, CA 10 years — **[proposed default, unsourced]**;
`DEGRADED{credential_expiring}` threshold 30 days reserved, not built (F-10). `ops/pki/README.md`:
CA key on removable media, passphrase-encrypted (`openssl ec -aes256`), passphrase held
separately; `ops/pki/out/` gitignored; scripts only in repo.

### 9.2 `docker-compose.sim.yml` (overlay; F-2)

```yaml
services:
  sim-init:
    build: ./sim
    command: ["simctl", "pki", "init", "--out", "/sim-pki", "--principals", "rmc-01,backend-ingest,harness,broker"]
    volumes: [sim-pki:/sim-pki]
  mosquitto:
    secrets: [sim_broker_ca_cert, sim_broker_server_cert, sim_broker_server_key, sim_broker_crl]
    volumes: [mosquitto-data-sim:/mosquitto/data, mosquitto-acl-sim:/mosquitto/acl:ro]      # distinct volumes: prod broker state untouched
    configs: [{source: mosquitto_sim_conf, target: /mosquitto/config/mosquitto.conf}]         # same file but cafile → sim CA ONLY (replaces, never appends)
  backend:
    environment: {TROPHIC_SIM_OVERLAY: "1", TROPHIC_ACL_EXTRA: /overlay/harness.acl, TROPHIC_SMTP_HOST: mailsink}
    secrets: [sim_broker_ca_cert, sim_backend_client_cert, sim_backend_client_key]
    volumes: [mosquitto-acl-sim:/mosquitto/acl, ./sim/overlay/harness.acl:/overlay/harness.acl:ro]
  virtual-room:
    build: ./sim
    environment: {SCENARIO: /scenarios/${SCENARIO:-short-soak}.yaml, RACKS: "${RACKS:-11}"}
    volumes: [sim-pki:/sim-pki:ro, sim-state:/state]
    networks: [default, simctl-net]
    # no ports:
  mailsink: {image: axllent/mailpit@sha256:<pinned>, networks: [default]}   # AC-ALR-04a; UI not published
networks: {simctl-net: {internal: true}}
volumes: {sim-pki: {}, sim-state: {}, mosquitto-data-sim: {}, mosquitto-acl-sim: {}}
```

Guarantees: prod file never mentions `sim`, `simctl`, `harness` or `override` (lint in
`boundary-check`); the overlay's broker trusts the sim CA **only**; the harness principal
exists only in `sim/overlay/harness.acl`, appended only when `TROPHIC_SIM_OVERLAY=1`;
overlay uses its own broker volumes; `docs/ops/sim-on-host.md`: after any overlay run on
the facility host, `docker compose -f docker-compose.prod.yml up -d --force-recreate
mosquitto backend` (prod secrets, ACL re-materialised, sim volumes untouched and unused);
e2e asserts a sim-CA cert is refused by the prod-configured broker (§10.6).

### 9.3 `.github/workflows/build-test-deploy.yml` jobs

| Job | Steps |
|---|---|
| `detect` | probe `backend/go.mod`, `frontend/package.json`, `firmware/rack-controller/CMakeLists.txt`, `sim/go.mod`; outputs `backend, frontend, firmware, sim` |
| `schema` | `npx ajv-cli@5 compile -s 'docs/iot/schema/v1/**/*.schema.json'`; validate every golden against its schema; `npx @redocly/cli lint docs/api/openapi.v1.yaml`; assert the only `commands` action const is `sys.identify` |
| `backend` | `go-version-file: backend/go.mod`; services: `timescale/timescaledb:2.17.2-pg16`; `go build -mod=readonly ./...`, `go vet`, `govulncheck ./...`, `go test ./...` (includes `repotest` gate, `spectest`, conformance, audit immutability, alert machine tests); `oapi-codegen` regenerate `internal/api/gen` → `git diff --exit-code` |
| `frontend` | Node 20 pinned; `npm ci`; `npm audit --audit-level=high`; `npm run api:generate` → `git diff --exit-code src/api/generated`; `npm run lint`; `npm test`; `npm run build` |
| `firmware` | container `espressif/idf:v5.3.1@sha256:<pinned>`; `cd firmware/rack-controller`; host conformance test; relay-token grep; `idf.py build`; symbol check (§7.5) |
| `sim` | `go build/vet/govulncheck/test ./...`; conformance; scenario loader tests (soak refuses override); `virtual-room --scenario short-soak --time-scale 60 --duration 10m --broker embedded` short soak against an in-process test broker (`mochi-mqtt/server`), asserts ring/replay/gap events |
| `boundary-check` | script: no `require`/`replace` between `backend/go.mod` and `sim/go.mod`; no import path crossing `backend/`↔`sim/`; grep for `../backend|../sim|../firmware|../frontend` in source across subsystems; prod compose contains no `sim`/`simctl`/`harness`; `frontend/package-lock.json` has no `mqtt` |
| `secrets-scan` | `gitleaks detect --redact` (required) |
| `e2e-sim` | `docker compose -f docker-compose.prod.yml -f docker-compose.sim.yml up`; playwright + Go harness: `short-soak` (10 min scaled), `reject-path`, `commissioning`, `safety-override`, `export-restore` (second clean stack via `-p drill`), `home-5s`, `acl-scoping`, `sim-ca-refused-by-prod` |
| `deploy` | on push to `main` after all: SSH with a **forced command** (`/opt/trophic/deploy.sh`) as a dedicated `deploy` user, host allow-list on sshd, environment `production` with required reviewers; script does `git pull && docker compose -f docker-compose.prod.yml build && up -d && docker compose exec backend /trophic migrate` (F-15 hardening; pull-based deploy is the R2 target) |
| `contracts` (when added) | `git submodule update --init contracts` pinned to tag `v0.2.0`; `backend`/`sim` tests read `contracts/sensors/instrument-tags.md` expected counts from the checkout when present, else from an embedded copy with the same sha (drift check) |

---

## 10. Test plan skeleton (for `automation-engineer` / `qa-e2e-validator`)

### 10.1 72 h soak with gap accounting (deploy host, overlay, `72h-soak.yaml`)

Pre: fresh overlay stack, 11 racks + rmc-01 + 3 I/O nodes, all claimed and active via the
`commissioning` harness. During: timeline of §6.7. Post-report (`sim/cmd/simctl report
--out docs/pipeline/phase-1-foundation/soak-report.md`): (1) for every `(device, boot_id)`
published range, `stored seq ∪ gap rows = full range`; (2) `telemetry_gaps.status='open' = 0`;
(3) every `unfillable` row has `reason ∈ {buffer_overflow, link_gap, not_buffered}`; (4)
`backend-only` step produced ≥1 `ingest_outages{startup}` row and **zero** gap rows (broker
queue held them); (5) `edge-ungraceful` produced LWT → `health{offline,lwt}`, inherited
OFFLINE on 14 nodes, **one** alert row per device (not per metric), replay with
`backfill=true`, gaps → `filled`, no alert evaluated on backfilled rows (assert
`alert_eval_state.last_sample_ts` never moves during replay); (6) `metric stale` produced
exactly one `device.stale_telemetry` alert citing `design_parameter`, resolved on the first
new sample; (7) `range` burst rows are `implausible-rejected`, absent from LKS and rollups'
`avg`, present in `n_rejected`; (8) `telemetry_1m`/`_1h` have no missing buckets for online
devices; (9) `platform_health.ingest_lag_s` not monotonic; (10) `alerts.severity='SAFETY'`
count = 0; (11) each nightly `climate.rh_band` alert auto-resolved before the next lights-off;
(12) `te02` crossed 26 °C → `water.solution_temp_operating_max` raised and resolved; 28 never
crossed. Real-hardware variant: same run with `--serial` and the skeleton attached; add
`sys.heap_free` non-decreasing-trend check.

### 10.2 Commissioning round-trip incl. REJECT (e2e-sim `commissioning`, `reject-path`)

Two browser contexts (technician `tech1`, operator `op1`). Steps 0–5 against `rc-01`; assert
`commissioning_steps` = 6 rows with actors; `commands` row acked SUCCESS with latency;
`identify_confirmations.confirmed_by_user_id = op1`. Negative: `op1` `POST claim` → 403;
claim of a bus node before `rmc-01` is claimed → 409 `parent_unclaimed`; fixture
`rc-badfailsafe` (fill advertised `OPEN`) blocked at step 2 (412, red FAULT text quotes the
key path); fixture `rc-unknownclass` shows "recorded, not rendered"; `duration_s: 600` →
edge `params_out_of_range` (and backend 412 fast path — tested by bypassing the console).
Harness principal (sim CA): publish `cmd{co2.enrich}` to `rc-02` → `ack{REJECTED,
capability_not_advertised}`; `cmd{light.dim, set, {level: 70}}` → `action_not_supported`;
`cmd{control.compute, sys.backfill}` → `action_not_supported`; duplicate `cmd_id` → stored
ack with `duplicate: true`; `cmd_seq` regression → `stale_sequence`; cmd to an unclaimed
fixture → `device_unclaimed`. Backend static test AC-DEV-03e passes (§5.6).

### 10.3 Home < 5 s (e2e-sim `home-5s`)

Fixture: 11 racks with 72 h seeded history (loader writes `telemetry_raw` directly, runs
aggregate refresh). Playwright with a 4G throttling profile (`downloadThroughput 4 Mbps,
latency 150 ms` — [proposed default]): cold load `/` after login; assert verdict band and its
`as_of` painted (`data-testid=verdict`) within **2 s** at p95 over 20 runs (p50 reported);
tier route within 3 s; every numeric node has a sibling `[data-age]` and `[data-provenance]`;
VPD tile has `CALC` + `formula_version`; inject `override` → verdict flips to `safety` within
`poll + 0 s`; `docker stop backend` → verdict `unknown`, no `ALL CLEAR` text, values dimmed,
service worker served nothing under `/api/`.

### 10.4 Org-scoping suite

`repotest` gate (§5.9) with a two-org fixture; API test: every operation in the spec called
under org A's session with org B ids → 404 (never 403, never data); `spectest` asserts no
`org` parameter; Mosquitto test (`acl-scoping`): a second sim principal `rmc-02` in
`room-2` publishes to `…/room-1/…` → broker refuses (assert no message received by a
`trophic/#` observer); `rmc-01` publishes to `…/room-1/rack-01/rc-01/cmd` → refused (F-11);
`backend-ingest` publishes to a `telemetry/` leaf → refused.

### 10.5 Export/restore drill (e2e-sim `export-restore`)

After the short soak: `POST /backups/export` → download; tamper one byte → `restore --validate`
refuses naming the file; edit `manifest.db_schema_version` upward → refused with reason.
Second clean stack (`-p drill`, fresh volumes, **same** `totp_key` and `backup_key` files,
fresh everything else) → `migrate` → `restore --validate` ok → `restore --policy import-as-new`
→ counts = manifest; `audit_log` count = manifest and `UPDATE audit_log` fails both ways;
`telemetry_1m` rows present; `tech1` **logs in with TOTP** (F-3); `MaterialiseACL` output
equals the original aclfile; virtual devices reconnect with the same certs and publish;
`telemetry_gaps` row with cause `backend-down` covers the restore window; `import_runs` and
two audit rows (export, restore) exist.

### 10.6 Security pass (inputs to `security-safety-reviewer`)

Automated: gitleaks; `govulncheck`; `npm audit`; `InsecureSkipVerify` grep; relay-token grep;
symbol check; `sim-ca-refused-by-prod` (bring up prod compose with prod-shaped test secrets,
connect with a sim-CA cert → TLS handshake failure); `PATCH /alert-rules/{safety}` → 403;
`POST ack` without action code → 412; resolve during `monitor_silent` → 412; login error
identical for wrong password vs wrong TOTP; 6th login attempt/min → 429; cookie flags;
CSP header present; `X-Requested-With` missing → 403 on POST; unclaimed device command → 409
and nothing on the broker (observer). Manual: ops/pki README, restore-drill runbook, overlay
teardown note, deploy forced-command config.

---

## 11. Defaults and unknowns

### 11.1 Every [proposed default, unsourced] value and where it is configured

| Value | Default | Configured at |
|---|---|---|
| `interval_s` all classes | 30 | profile JSON `interval_s`; sim `fixture.interval_s` |
| Heartbeat | 30 s | `vroom` `--heartbeat-s`; `health.heartbeat_s` on the wire |
| `offline_after_s` | 90 | backend `TROPHIC_OFFLINE_AFTER_S` (default from `health.heartbeat_s × 3` when present) |
| `link_timeout_s` | 10 | `vroom --link-timeout-s` |
| Staleness floors | T/RH 120, CO2 300, switches/state 120, water-side 120 | `ingest.Staleness.Floors` (config `TROPHIC_STALE_FLOOR_<class>`) |
| Stuck-value N | 20 | `TROPHIC_STUCK_N` |
| `sys.identify.max_duration_s` | 60 | profile `control.compute.params`; firmware `CONFIG_TROPHIC_IDENTIFY_MAX_S` |
| `ack_timeout_s / max_retries / ttl_s / backend_wait_s / dedup ring` | 5 / 2 / 60 / 30 / 32 | `vroom` flags; backend `Publisher` consts; firmware `CONFIG_TROPHIC_CMD_RING` |
| Ring capacity | 27 000 (6 h) | `vroom --ring-capacity`; reported in `health.buffer.capacity` |
| Broker `max_queued_messages` | 60 000 | `ops/mosquitto/mosquitto.conf` |
| `telemetry_dedup` prune | 48 h | migration comment + `TROPHIC_DEDUP_TTL` |
| VPD pairing window | 10 s | `ingest.VPDPairer.Window` |
| Air-T plausibility (virtual) | −40..85 °C "project-doc placeholder" | profile `range` + `range_note` |
| RH plausibility | 0–100 | profile |
| Enrichment NDIR range | 400–5000 | profile |
| Safety-NDIR class fallback | 0–10 000 ppm | `class_fallback_ranges(sense.co2, co2_ppm)` note "class fallback, part range unknown" |
| pH / EC ranges | 0–14 / 0–5 (hi open: fallback hi = 20 mS/cm for the gate only, labelled) | profile; `class_fallback_ranges` |
| Water-temp class fallback | 0–100 °C "liquid-water physics bound, class fallback — not an instrument range" | `class_fallback_ranges(sense.water_temp, water_temp_c)` |
| Band rule `duration_s/clear_s` | 300/300 (climate, CO2), 120/120 (solution T 26), 600/600 (pH), 90/90 (fan stopped); SAFETY 0/120 | `alerts/seed.go`, `duration_label: unsourced-default`; per-zone override |
| RH hysteresis semantics | crossed bound ± 5, `hysteresis_note` "borrowed from control" | `alert_rules.hysteresis` |
| Platform ingest-lag threshold; broker-disconnected | 10 s; 60 s | `TROPHIC_LAG_THRESHOLD_S`, rule `platform.broker_disconnected` |
| NOISY tags | ≥10 acks w/o action or ≥20 auto-resolved <5 min per 30 d | `alerts/noisy.go` consts |
| Export raw window | 30 d | `export --raw-window-days`, `POST /backups/export` |
| Scheduled backup | 02:00 facility tz, retention 14 | `TROPHIC_BACKUP_CRON`, `TROPHIC_BACKUP_RETAIN` |
| Session expiry | idle 12 h, absolute 7 d | `TROPHIC_SESSION_IDLE_H`, `_ABS_D` |
| Login rate limit | 5/min/email, 20/min/IP | `authz/ratelimit.go` |
| argon2id params | t=3, m=64 MiB, p=4 | `authz/password.go` |
| Cert validity | device 2 y, service 1 y, CA 10 y; expiring warn 30 d (reserved) | `ops/pki/*.sh` |
| Console polling / backoff | §8.1 table; ×2 to 60 s | `composables/usePolling.ts` |
| Chart cut-over | raw ≤ 6 h, 1m ≤ 7 d, else 1h; raw cap 24 h | `api/handlers_telemetry.go` |
| Load fixture | 13 racks (52 zones) | `load-52-zones.yaml` |
| Home budget percentile | p95 (p50 reported) | `home-5s` spec |
| Sim diurnal params | §6.5 table | `sim_params` in scenario; echoed in report |
| Contact debounce (firmware) | 50 ms | `CONFIG_TROPHIC_CONTACT_DEBOUNCE_MS` |
| Mosquitto ACL reload poll | 5 s | `ops/mosquitto/entrypoint.sh` |

### 11.2 Every [unknown] and what the code does without it

| Unknown | Behaviour |
|---|---|
| Air-T sensor part/range | virtual profile carries the placeholder range; real skeleton omits `range` unless the driver states it → class fallback recorded on every row |
| Water-temp element part/range | class fallback 0–100 °C recorded; row `range_source = class_fallback` |
| CO2 safety monitor part/range | profile omits `range`; fallback 0–10 000; bound invariant satisfied; H-review item |
| Fan model, pulses/rev, low-fan band | metric is `fan_tacho_hz`; no rpm anywhere; only `== 0` rule exists; no "tacho low" row or badge |
| EC operating target | no rule row; `ValueEnvelope` for EC has no band state; console shows no in/out badge; sim label `unsourced-sim-placeholder` |
| G1 gate | not referenced by any code path |
| DLI target | `dli_mol_m2_day` series exists only when PPFD entered; no rule, no badge; null never zero |
| Alert duration windows | LLD defaults above, `duration_label = unsourced-default`, per-zone override available |
| Solution-T low bound; CO2 ambient floor | no rules; sim ambient floor is a `sim_param` |
| Room-vs-rack divergence tolerance | `device_events{kind: cross_check_divergence}` **not** emitted (no tolerance) — a counter of pairs observed only |
| Rate-of-change limits | gate absent; `quality_reason = rate_of_change` never produced |
| Leak/LSH-04 wiring | firmware reads GPIO as telemetry only; no interlock logic |
| Relay polarity/strapping; fan/LED de-energised states | no relay pin exists in the image; §7.1 gate |
| Modbus register map | `rack-link/0` only; `reg_image` internal, unpublished |
| ESP32 variant/framework | board config required; ESP-IDF chosen (LLD); `TROPHIC_BOARD` mandatory |
| GROW vs GROW-S first racks | affects only the optional real-hardware soak variant |
| io-room E-stop / floor flood | classes omitted from `io-room-v0.json`; rules seeded but evaluate against nothing (no scope resolves) |
| Leak/LSH-04 debounce | 50 ms firmware default, labelled non-safety |
| Offsite backup target | `backups` volume only; OQ-9 |
| LED visibility through enclosure | console renders `identify.active` from health regardless |

---

## 12. Security-review findings — disposition in this LLD

| F | Where resolved |
|---|---|
| F-2 | §9.2 (sim CA only; overlay-only ACL file appended only under `TROPHIC_SIM_OVERLAY=1`; distinct sim volumes; `sim-on-host.md`; `sim-ca-refused-by-prod` e2e) |
| F-3 / F-12 | §3.3 (`sessions`, `login_attempts`, `totp_enrolments` excluded; `totp_key` split; `session_key` removed; `requires_secrets` in manifest); §10.5 logs in with TOTP |
| F-5 | §3.7 comparators `> 26`, `> 28`, `>= 5000` with `comparator_basis` |
| F-6 | §3.6 `CHECK`; §4.2 `PATCH` → 403 `safety_rule_immutable`; seed asserts SAFETY `enabled=true` |
| F-7 | §5.5 state table + named tests; `monitor_silent_since` on the alert and in the API |
| F-8 | §7.1 verbatim conditions, build-time assertions, `_Static_assert` on the LED pin |
| F-9 | §6.7 unix socket / internal network, no host port, compose lint |
| F-10 | §9.1 TLS parameters, CA issuance, validity, key handling |
| F-11 | §2.1 leaf-enumerated write ACL (no `cmd`/`state/desired` for the room controller); §10.4 test |
| F-13 | §8.6 and §9.3 (lockfile, `npm ci`, `ignore-scripts`, audit, `-mod=readonly`, `govulncheck`, pinned toolchains and image digests) |
| F-14 | §2.7 verbatim statement in `rack-link-0.md` |
| F-15 | §9.3 deploy row (forced command, dedicated user, environment reviewers); pull-based deploy recorded as R2 |

---

## 13. Flags

**To `systems-integration-reviewer` (constraint-level, none loosened):** (1) Mosquitto's
aclfile has no deny rule, so F-11 is met by enumerating write leaves rather than a deny —
same effect, different mechanism; (2) `sys.ping` dropped from the backend publisher
allow-list (no Phase 1 caller; brief §10.1 C(c)); (3) the RH ADVISORY with the borrowed 5 %
hysteresis clears only when RH ≤ 65 % — by day, not at the end of the night excursion; the
soak asserts daily resolution rather than immediate clearance; (4) `session_key` removed
from the secret set (opaque sessions need no signing key); `totp_key` added; (5) `bytes` added
to the unit enum for `sys.heap_free` (review agenda); (6) TimescaleDB cannot enforce a unique
index without the time column, so dedup on `(org, device_id, boot_id, seq)` is a separate
`telemetry_dedup` table with a 48 h window plus an in-memory ring — a replay older than 48 h
would not be deduplicated, which is unreachable with a 6 h ring but is stated.

**To `security-safety-reviewer`:** §3.7 comparators; §5.5 SAFETY admission/resolve tests;
§7.1 gate text; §9.1/9.2 trust anchors; the `sim_test_injection` field path (sim-side,
outside the schema's enumerated properties, copied to `alerts.is_test_injection`).

**To `hld-architect` / hardware review agenda:** `bytes` unit; per-rack `LSH-04` instance
convention; terrace pH/EC untagged.
