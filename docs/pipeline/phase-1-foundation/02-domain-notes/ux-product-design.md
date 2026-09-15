# 02 — Domain note: UX & product design (operator console v0)

**Author:** ux-product-designer · **Date:** 2026-09-10 · **Pipeline:** `/ship phase-1-foundation`
**Scope:** Phase 1 deliverable 2 — operator console v0: Login, Home, Grow, Devices, Alerts,
Settings. **No control surfaces.** Nothing on any screen sends a command to a valve, LED,
fan, pump, or dosing unit. The one command the console issues in Phase 1 is the
commissioning `identify` no-op (§5).

**Inputs read:** `01-classification.md`; `docs/roadmap/release-plan.md` (Phase 1 lock);
`docs/ux/information-architecture.md`; `docs/requirements/requirements.md` (FR-FAC, FR-DEV,
FR-TEL, FR-ALR, FR-USR, FR-BKP, NFR-UX/PER); `docs/adr/0002`, `0009`;
`docs/pipeline/frontend-framework-adr/03..05`; `docs/iot/device-control-model.md` §1–§10;
`docs/security/security-architecture.md` §2, §5; `docs/data/rd-data-model.md` §2;
`docs/decisions/open-questions-risks-next-actions.md` (AR-8, AR-12);
`.github/agentic-rules/safety-rules.json`; `trophic-contracts v0.2.0`
(`capabilities/absent-by-design.md`, `sensors/instrument-tags.md`, `conventions/units.md`).

This note is the pre-wireframe layer (IA §6). Everything below is low-fidelity by design;
no visual design is authorized by it.

---

## 1. Personas served in Phase 1 and their top tasks

| Persona | Role (security-architecture §2) | Served in Phase 1? | Top tasks in Phase 1 |
|---|---|---|---|
| **P1 Operator/maintainer** (Ooty, primary) | `operator` | **Yes — primary** | T1 "Is anything wrong?" on entering the room, phone in hand (UC-01). T2 Triage an alert: what, where, since when, has hardware already acted, record what I did (UC-02). T3 Look at one rack/tier: what the sensors say right now and for the last hours (UC-03). T4 Confirm a device is alive / see why it dropped. T5 Confirm the last backup ran. |
| **P4 Device technician** (Trophic; Coimbatore, often remote) | `technician` | **Yes** | T6 Commission a device end-to-end with someone else standing at the rack (UC-04). T7 Fleet health: which devices are offline/stale/degraded and since when (UC-19, read-only part). T8 Read a device's advertised capability profile and firmware version. |
| P2 Grow manager | `grower` | Only as a login | No recipes/batches in Phase 1. Sees the same screens as P1. Nothing is designed for P2 here. |
| P3 Owner | `viewer` | Only as a login | Read-only view of the same screens. No analytics. |
| `admin` | — | Yes | T9 Create users, assign roles, reset TOTP. T10 Run/verify export; validate a restore bundle. |
| P5 customer, P6 external farm, P7 aquarium | — | **No** | Not a surface in Phase 1; nothing on these screens may reference orders, customers, tanks, or tenants. |

Two things this table forces on the design:

- **P1 and P4 are usually two people in two places during commissioning.** The
  technician drives from a desktop in Coimbatore; the operator in Ooty powers the board on
  and reads the label. The commissioning flow (§5) is therefore designed as a two-party
  flow, not a single-user wizard, and the on-site confirmation step records *who* confirmed.
- **P1 is the same person as P2 at this facility size.** Role separation must never block
  a task; the `grower` role simply sees nothing extra in Phase 1.

Design constraint applied to every screen: *a person standing in the room, with wet
hands and marginal Wi-Fi, can still get the answer.* Concretely: primary touch targets are
large and single-tap; no hover-only affordances; no drag gestures on any required path;
every screen states data age and connection state; the app shell loads offline and says
so.

---

## 2. Information architecture and navigation (Phase 1 subset)

The navigation is the IA §3 tree with the unbuilt sections **absent**, not greyed out. A
greyed "Batches" entry is a promise the release does not keep; a missing one is honest.
The IA document remains the map; Phase 1 registers only these routes:

```
/login
/                     Home  (facility status)
/grow                 Grow  → /grow/rooms/:room → /grow/racks/:rack → /grow/tiers/:tier
/devices              Devices → /devices/:id → /devices/commission (wizard)
/alerts               Alerts  → /alerts/:id ; tabs: Active | History | Rules
/settings             Settings → /settings/users | /settings/backup | /settings/system
                                 | /settings/facility (see flag F-1) | /settings/safety-envelope
```

**App shell (present on every authenticated route):**

```
+------------------------------------------------------------------+
| [SAFETY BANNER — only while a SAFETY alert is active; full-width, |
|  cannot be closed; tap → /alerts/:id]                             |
+------------------------------------------------------------------+
| [CONNECTION STRIP: "Live · as of 09:41:12" | "DISCONNECTED since  |
|  09:38 — showing last received 09:37:50" | "Backend reachable,    |
|  room controller not seen for 4m 10s"]                            |
+------------------------------------------------------------------+
|                                                                  |
|                      route content                               |
|                                                                  |
+------------------------------------------------------------------+
| [Home]   [Grow]   [Devices]   [Alerts (n)]   [Settings]          |   ← bottom tab bar on
+------------------------------------------------------------------+     phone; left rail on desktop
```

Shell rules the LLD must implement, not the screens:

1. **Alerts tab badge** = count of active alerts, SAFETY count rendered separately and
   first ("1 SAFETY · 3"). Badge counts come from the same summary endpoint Home uses, so
   they cannot disagree with Home.
2. **SAFETY banner** lives in the shell, outside route content, above the connection strip.
   It carries: severity word, rule name, zone/device path, duration, and the sentence
   "Hardware interlock state — the console mirrors this; it does not make the room safe"
   when the rule is an interlock mirror. One tap opens the alert. No close control exists.
3. **Connection strip** is always visible. It distinguishes three facts that operators
   conflate: browser ↔ backend reachable; backend ↔ broker/ingest healthy; room controller
   last seen. All three come from the summary endpoint plus the client's own fetch result.
4. **Disconnected state** (per ADR-0009 PWA boundary): the shell renders, all numbers
   dim, ages keep counting up from the last successful response, and Home's verdict
   becomes `UNKNOWN`. Nothing served from cache is ever rendered as current.
5. **Sim marker.** Phase 1 runs on virtual devices (`sim: true`, device-control-model §10).
   The shell shows a persistent "SIMULATED: n of m devices are virtual" chip when any
   device in the org is virtual, and every virtual device/tier/rack carries a `SIM` tag
   inline. This is a provenance rule (§4), not a dev-mode flag.
6. **Role gating** hides write affordances (claim, assign, ack, resolve, user CRUD,
   export) for roles that lack them; it never hides data. Server enforces; client mirrors.
7. **Time**: facility-local (IST) everywhere; tapping any timestamp reveals UTC
   (NFR-UX-03).

---

## 3. Per-screen low-fidelity wireframes and data contracts

Each screen lists the elements and the data every element needs. Endpoint names are hints
for the LLD's OpenAPI spec (the contract of record, ADR-0008 rule 1), not decisions.

### 3.1 Login

```
+----------------------------------+
|  Trophic CEA — Ooty              |
|                                  |
|  Email     [__________________]  |
|  Password  [__________________]  |
|  TOTP code [______]              |
|                                  |
|  [        Sign in        ]       |
|                                  |
|  (error: "Sign-in failed." —     |
|   never says which factor)       |
|                                  |
|  Offline: "No connection to the  |
|  facility host. Hardware safety  |
|  does not depend on this app."   |
+----------------------------------+
```

- Single form, all three factors on one screen (fewer round-trips on poor connectivity;
  TOTP is mandatory from day one per security-architecture §2, so there is no reason to
  stage it).
- Data: `POST /auth/login {email, password, totp}` → session cookie; `GET /session` →
  `{user_id, display_name, role, org_id, session_expires_at}`.
- After login, land on Home. Session expiry returns to Login with the route preserved.
- No "remember device", no password reset by email in Phase 1 (admin resets; see 3.6).

### 3.2 Home — facility status (UC-01, FR-FAC-04)

```
+------------------------------------------------------------------+
| VERDICT BAND (full width, largest text on screen)                 |
|  "ALL CLEAR"  |  "2 ISSUES"  |  "1 SAFETY ALERT"  |  "UNKNOWN"     |
|  as of 09:41:12 (3 s ago)   ·   Room controller seen 12 s ago     |
+------------------------------------------------------------------+
| [SAFETY]   ▌ CO2 safety monitor (mirror) — Room 1 — 4m 12s        |   ← only if active,
|            ▌ tap → alert                                          |     always first
+------------------------------------------------------------------+
| ACTIVE ALERTS (n)                                  [open Alerts]  |
|  ▌OPERATIONAL  Stale telemetry — Rack 3 T/RH — 6m 40s             |
|  ▌ADVISORY     Fan tacho low — Rack 7 tier 2 — 1h 05m             |
+------------------------------------------------------------------+
| DEVICE ISSUES (n)                                  [open Devices] |
|  offline 1 · stale 2 · degraded 0 · fault 0 · unclaimed 1         |
+------------------------------------------------------------------+
| ROOM 1 SNAPSHOT                                      [open Grow]  |
|  T 22.4 °C (41 s)   RH 63 % (41 s)   VPD ≈0.81 kPa CALC (41 s)     |
|  CO2 1120 µmol/mol (55 s)      Water TE-02 21.8 °C (2m 10s)        |
|  Racks: 11 total · 11 active · 0 idle · 1 with alerts              |
+------------------------------------------------------------------+
| PLATFORM                                          [open Settings] |
|  Ingest lag 1.2 s · Last backup 02:00 OK (7h ago) · Broker OK      |
+------------------------------------------------------------------+
```

Data contract — one call, `GET /facility/summary`, returning:

| Element | Fields needed |
|---|---|
| Verdict band | `verdict ∈ {all_clear, issues, safety, unknown}` computed **server-side**; `as_of` (server time); `active_alert_counts {safety, operational, advisory}`; `room_controller_last_seen_at`, `room_controller_stale: bool` |
| SAFETY row(s) | for each active SAFETY alert: `alert_id, rule_name, path, raised_at, is_interlock_mirror` |
| Active alerts | top N non-SAFETY by severity then age: `alert_id, severity, rule_name, path, raised_at, ack_state` |
| Device issues | counts by health state `{online, offline, stale, degraded, fault, unclaimed, quarantined}` |
| Room snapshot | per headline metric: `value, unit, provenance, ts_source, age_s, quality, stale, formula_version?` (see §4) — room T/RH (room I/O node), room CO2 control NDIR, TE-02; rack counts by zone status (FR-FAC-03) |
| Platform | `ingest_lag_s, broker_ok, last_backup {finished_at, status}` (FR-USR-03, FR-BKP-03) |

What Home does **not** show in Phase 1, and must not fake: "batches due" (R3) and "zones
off-target vs desired" (R2; there is no desired state). The IA's "Today: what needs me"
list collapses to alarms + device issues + connectivity. When R2/R3 land, the rows are
added between ACTIVE ALERTS and DEVICE ISSUES; the layout reserves nothing for them now.

### 3.3 Grow — facility → room → rack → tier (UC-03, FR-FAC-01..05, FR-TEL-04)

Phase 1 has one room; the room level still exists as a route so the hierarchy is real,
but Grow opens directly on the room when there is exactly one.

**Room view (rack list):**

```
+------------------------------------------------------------------+
| Room 1 — Ooty            as of 09:41:12 (3 s ago)                 |
| Room: T 22.4 °C (41s) · RH 63 % (41s) · VPD ≈0.81 CALC · CO2 1120 |
| Water: TE-01 20.9 °C (3m) · TE-02 21.8 °C (2m) · AT-03 EC — (n/a) |
|        AT-04 pH — (n/a) · LT-02 91 % (3m)                         |
+------------------------------------------------------------------+
| RACKS (list; map deferred — FR-FAC-05 gives position as metadata)|
|  Rack 1  ● active   T 22.1 (35s)  RH 64  VPD ≈0.80   ▌0 alerts    |
|  Rack 2  ● active   T 22.3 (38s)  RH 62  VPD ≈0.85   ▌0 alerts    |
|  Rack 3  ◐ stale    T 22.0 (6m40s) ...              ▌1 OPERATIONAL|
|  Rack 7  ● active   ...                              ▌1 ADVISORY  |
|  Rack 11 ○ offline  — (last seen 22m)               ▌1 OPERATIONAL|
|  [SIM] tag on virtual racks                                       |
+------------------------------------------------------------------+
```

Note the T/RH column is **rack-level** (one sensor per rack at tier 3 — absent-by-design
row 4). It is displayed on the rack row and never duplicated into tier rows as if measured
there.

**Rack view:**

```
+------------------------------------------------------------------+
| Rack 3  ◐ stale    lifecycle: active     controller: RC-03 (SIM)  |
| Rack sensors (at tier 3): T 22.0 °C (6m 40s, STALE) RH 61 %      |
|   VPD ≈0.83 kPa CALC v1 (6m 40s, STALE)                          |
| Rack interlocks (hard-wired, mirrored): LD-03 leak: clear (40s)   |
|   LSH-04 header high: clear (40s)                                 |
| Alerts on this rack: ▌OPERATIONAL Stale telemetry — 6m 40s → open |
+------------------------------------------------------------------+
| TIER 4 (zone Z-3-4) ● active                                      |
|   Light: reported 70 % on (42s)   Fan: 1180 rpm tacho (42s)       |
|   Flood: reported idle, last cycle 07:12 (42s)                    |
| TIER 3 (zone Z-3-3) ● active  ...                                 |
| TIER 2 ...                                                        |
| TIER 1 ...                                                        |
+------------------------------------------------------------------+
| Events (last 24 h): 08:55 controller restarted · 07:12 flood done |
+------------------------------------------------------------------+
```

**Tier detail:**

```
+------------------------------------------------------------------+
| Rack 3 · Tier 2 · zone Z-3-2  ● active    reason: —               |
| Rack-level environment (measured at tier 3, not here):            |
|   T 22.0 (6m 40s STALE) · RH 61 · VPD ≈0.83 CALC                  |
| This tier reports: light 70 % on (42s) · fan 1180 rpm (42s) ·     |
|   flood idle (42s)                                                |
+------------------------------------------------------------------+
| CHART  [T/RH rack] [fan tacho] [light state] [flood events]       |
|        range: 1h | 6h | 24h | 7d   resolution auto (raw/1m/1h)    |
|        rejected/suspect samples drawn as hollow markers, legend   |
|        says "suspect" / "rejected"                                |
+------------------------------------------------------------------+
| Devices serving this tier: RC-03 ch 2 (light.dim, airflow.speed,  |
|   irrigation.flood, valve.control ×2) → device detail             |
| Recent events for this zone (quality flags, reboots, gaps)        |
+------------------------------------------------------------------+
```

No "Desired" column is rendered anywhere in Phase 1. IA principle 3 (desired and actual
together) applies once desired state exists (R2); an empty Desired column now would be a
fake affordance.

Data contract:

| Element | Endpoint hint | Fields |
|---|---|---|
| Hierarchy + status | `GET /facility/tree` | rooms → racks → tiers → zones with `id, name, lifecycle (installed/active/maintenance/retired), zone_status (active/idle/offline), status_reason, position_ref, sim, alert_counts` |
| Room snapshot | `GET /rooms/{id}/snapshot` | room-scoped metrics per §4 value envelope; water-side tags (TE-01, TE-02, AT-03, AT-04, LT-02, LSH-04 per rack) keyed by instrument tag |
| Rack snapshot | `GET /racks/{id}/snapshot` | rack-level T/RH/VPD; interlock mirror states (LD-nn, LSH-04); controller device id + health; per-tier reported states (`light.state_report`, `airflow.tacho`, flood reported state) |
| Tier/zone detail | `GET /zones/{id}/snapshot` | same as rack subset + capability list from the zone's devices (ADR-0002: union of profiles) |
| Charts | `GET /zones/{id}/telemetry?metric=&from=&to=&resolution=raw|1m|1h` | series of `{ts_source, value, quality, calibration_id}` and gap markers (NFR-DAT-02: gaps are recorded gaps) |
| Events | `GET /zones/{id}/events?since=` | `{ts, kind (boot, claim, capability_advert, fault, gap, quality), detail}` |

Absent-by-design assertion 1 is enforced at the *rendering* level: a metric row renders
only if a device serving that scope advertises the matching `sense.*` / `*.state_report`
capability. No per-tier CO2, no per-tray level, no fixed PPFD, no per-tier T/RH — ever —
unless a profile advertises it.

### 3.4 Devices — registry, health, commissioning (FR-DEV-01..04, UC-04, UC-19)

**Fleet list:**

```
+------------------------------------------------------------------+
| Devices        filter: [all|online|offline|stale|degraded|fault|  |
|                        unclaimed|quarantined]   [+ Commission]   |
+------------------------------------------------------------------+
| UNCLAIMED (1)  — untrusted, receives no commands                   |
|   esp32-a4f2… advertised 3m ago · profile v1 · fp 7c:1e…  [Claim] |
+------------------------------------------------------------------+
| RC-01  rack controller  Rack 1   ● online  seen 12s   fw 0.1.0 SIM |
| RC-03  rack controller  Rack 3   ◐ stale   seen 6m40s fw 0.1.0     |
| RC-11  rack controller  Rack 11  ○ offline seen 22m   fw 0.1.0     |
| RIO-01 room I/O node    Room 1   ● online  seen 20s                |
| WS-01  water-skid node  Skid     ● online  seen 31s                |
| TIO-01 terrace node     Terrace  ● online  seen 45s                |
+------------------------------------------------------------------+
```

**Device detail:**

```
+------------------------------------------------------------------+
| RC-03  rack controller   ◐ stale (since 09:34:32, 6m 40s)          |
| Identity: device_id · hw fingerprint · claimed 2026-09-10 by tech1 |
| Firmware 0.1.0 · profile schema v1 · commissioned 2026-09-10       |
| Assignment: Rack 3 · lifecycle active                              |
+------------------------------------------------------------------+
| CAPABILITIES (advertised)                                          |
|  light.dim {min 0, max 100} ×4 (tier 1..4)                         |
|  airflow.speed / airflow.tacho ×4                                   |
|  irrigation.flood ×4 · valve.control {failsafe: CLOSED} ×4 (fill)  |
|  valve.control {failsafe: OPEN} ×4 (drain) · sense.temp_rh ×1 (t3) |
|  sense.leak ×1 · unknown class "x.foo" v2 — recorded, not rendered |
+------------------------------------------------------------------+
| HEALTH: last telemetry 09:34:32 · last event 08:55 reboot ·        |
|   staleness threshold (from policy) 2 min · seq gap count 0        |
| COMMANDS (Phase 1: identify only)                                  |
|   09:12:04 identify → SUCCESS 0.41 s (confirmed on-site by op1)    |
| EVENTS (boot, claim, advert, faults, gaps)                         |
| [Identify]  (technician/admin only; the only command in Phase 1)   |
+------------------------------------------------------------------+
```

The fail-safe state shown per `valve.control` instance is the device's *advertised*
`failsafe_state`, displayed verbatim, cross-checked against
`safety-rules.json.valve_and_actuator_fail_safe_states`. A mismatch (device advertises
fill solenoid failsafe OPEN) is rendered as a **fault** with the source text quoted — it is
information the technician needs before activation, not something the console corrects.

Data contract:

| Element | Endpoint hint | Fields |
|---|---|---|
| Fleet list | `GET /devices?health=&claim_state=` | `id, label, type (descriptive only, ADR-0002 §4), assignment {room/rack/tier}, health (online/offline/stale/degraded/fault), last_seen_at, age_s, firmware, sim, claim_state (unclaimed/claimed/quarantined/revoked)` |
| Unclaimed | `GET /devices?claim_state=unclaimed` | `advertised_id, fingerprint, profile_schema_version, first_seen_at` |
| Detail | `GET /devices/{id}` | identity, fingerprint, claimed_by/at, firmware, profile schema version, assignment, lifecycle, commissioning record (every step + actor + timestamp, FR-DEV-03) |
| Capabilities | `GET /devices/{id}/capabilities` | parsed profile: list of `{class, version, params, instance (tier/channel), scope}` + `unrecognized[]` (ADR-0002 §3) |
| Health | part of detail | `last_telemetry_at, last_event_at, staleness_max_age_s (policy), seq_gaps, degraded_reason` |
| Commands | `GET /devices/{id}/commands` | `{command_id, kind, issued_by, issued_at, ack {result, reason, latency_ms, state_after}, confirmed_by?, confirmed_at?}` |
| Events | `GET /devices/{id}/events` | as zone events |

Commissioning wizard is §5.

### 3.5 Alerts — center (FR-ALR-01..04, 06; UC-02)

**Active tab:**

```
+------------------------------------------------------------------+
| Alerts   [Active (4)] [History] [Rules]                            |
+------------------------------------------------------------------+
| ▌SAFETY  (pinned; cannot be dismissed)                             |
|  CO2 safety monitor ≥ 5000 ppm (MIRROR of hardware alarm)          |
|  Room 1 · since 09:37:20 (4m 12s) · now 5240 ppm (8s) · ack: no    |
|  Hardware state: fail-closed CO2 solenoid — from device event      |
+------------------------------------------------------------------+
| ▌OPERATIONAL  Stale telemetry — Rack 3 T/RH — 6m 40s · ack: no     |
| ▌OPERATIONAL  Device offline — RC-11 — 22m · ack: yes (op1 09:25)  |
| ▌ADVISORY     Fan tacho low — Rack 7 tier 2 — 1h 05m · ack: no     |
+------------------------------------------------------------------+
| sort: severity, then age. Grouped: one row per rule × scope.       |
+------------------------------------------------------------------+
```

**Alert detail (SAFETY example):**

```
+------------------------------------------------------------------+
| ▌SAFETY   CO2 safety monitor ≥ 5000 ppm                            |
| Room 1 · NDIR @ 300 mm (safety sensor) · raised 09:37:20 (4m 12s)  |
| Current 5240 ppm (8 s ago) vs threshold 5000 ppm                   |
|                                                                    |
| WHAT THE HARDWARE HAS DONE                                         |
|  Independent CO2 monitor, fail-closed solenoid. This console       |
|  mirrors the alarm; it did not cause the solenoid to close and     |
|  cannot open it. (source: safety-rules.json climate.co2_safety_    |
|  alarm_ppm — CEA_Room_-_Floor_Plan.ocr.md §06)                     |
|                                                                    |
| WHAT THIS RULE PROTECTS   (FR-ALR-06 link → Rules/{id})           |
|  "A 22 kg CO2 cylinder released into a 60.2 m3 room …" (quoted)    |
|                                                                    |
| TIMELINE                                                           |
|  09:37:20 raised · 09:37:21 notified (console) · —                 |
|                                                                    |
| [ Acknowledge — record action taken ]   (no "dismiss" exists)      |
|   action: (o) Ventilated room and left  (o) Checked cylinder line  |
|           (o) Other: [________________]   note: [____________]      |
| [ Resolve ] — disabled: "condition still present (5240 ppm, 8 s)"  |
+------------------------------------------------------------------+
```

**History tab:** same rows, filter by severity/rule/scope/date; each row shows
raise → ack (who, action) → resolve (who, reason) with durations. **Rules tab:** list of
rule definitions: `id, name, severity, scope (per-zone/rack/room), threshold, duration,
source citation (verbatim), enabled`; SAFETY rows are read-only with the citation shown;
non-SAFETY per-zone thresholds editable by `grower`/`admin` (flag F-3).

Data contract:

| Element | Endpoint hint | Fields |
|---|---|---|
| Active list | `GET /alerts?state=active` | `id, severity, rule_id, rule_name, scope_path, raised_at, duration_s, current_value {value, unit, ts_source, age_s}, threshold, ack {by, at, action}, is_interlock_mirror, hardware_state_text?` |
| Detail | `GET /alerts/{id}` | above + `timeline[]`, `rule {definition, source_citation, protects_text}`, `resolvable: bool, resolve_blocked_reason` |
| Ack | `POST /alerts/{id}/ack {action_code, note}` | SAFETY: `action_code` required; server rejects empty |
| Resolve | `POST /alerts/{id}/resolve {reason_code, note}` | server refuses for SAFETY while condition present |
| History | `GET /alerts?state=resolved&from=&to=&severity=&rule_id=&scope=` | paginated |
| Rules | `GET /alert-rules`, `GET /alert-rules/{id}`, `PATCH /alert-rules/{id}` (non-SAFETY scope overrides only) | `id, name, severity, scope, threshold, duration_s, source_citation, editable: bool, noisy_stats {acks_without_action_30d}` |

Behavioural rules are in §6.

### 3.6 Settings — users, backup/export, system (FR-USR-01..03, FR-BKP-01..03)

```
+------------------------------------------------------------------+
| Settings   [Users] [Backup & export] [System] [Facility*] [Safety  |
|            envelope]                                               |
+------------------------------------------------------------------+
| USERS (admin only)                                                 |
|  name · email · role · TOTP enrolled · last sign-in · [disable]    |
|  [+ Add user] → email, display name, role; server issues a         |
|   one-time enrolment link/QR for TOTP; [Reset TOTP] per user       |
+------------------------------------------------------------------+
| BACKUP & EXPORT (J7)                                               |
|  Last scheduled backup: 2026-09-10 02:00 OK · 412 MB · sha256 …    |
|  Schedule: daily 02:00 · retention 14 (read-only in v0)            |
|  [ Export now ]  → progress → download bundle (schema v, manifest) |
|  Recent bundles: list with size, checksum, [download]              |
|  RESTORE / IMPORT: [Upload bundle] → validation report (schema     |
|   version, checksum verify, entity counts, conflicts) →            |
|   policy: (o) import as new  (o) replace — with warnings →         |
|   type facility name to confirm → run → verification report       |
+------------------------------------------------------------------+
| SYSTEM (FR-USR-03)                                                 |
|  backend version · broker OK · ingest lag 1.2 s · storage 38 %     |
|  · last restart 3d · MQTT clients: room controller seen 12 s       |
+------------------------------------------------------------------+
| SAFETY ENVELOPE (read-only; source safety-rules.json + citation)   |
|  the table alert citations link into; TBD entries shown as "TBD —  |
|  needs domain expert input", never as a number                     |
+------------------------------------------------------------------+
```

Data contract:

| Element | Endpoint hint |
|---|---|
| Users | `GET /users`, `POST /users`, `PATCH /users/{id} {role, disabled}`, `POST /users/{id}/totp/reset` |
| Backup | `GET /backups` (schedule, last run, list), `POST /backups/export` → job id, `GET /backups/jobs/{id}`, `GET /backups/{id}/download` |
| Restore | `POST /backups/import/validate` (multipart) → report; `POST /backups/import {bundle_id, policy}` → job; `GET /backups/jobs/{id}` |
| System | `GET /system/health` |
| Safety envelope | `GET /safety-envelope` (server serves the parsed JSON with citations; the console never bundles the file) |
| Facility (flag F-1) | `GET/POST/PATCH /rooms`, `/racks`, `/tiers` — minimal admin CRUD if the HLD does not seed structure from config |

The Facility tab is marked `*` because it is not in the six-screen lock; see F-1.

---

## 4. Number rendering rules: source age (release-plan acceptance) and provenance (AR-12)

These two rules are one rendering component, used everywhere a number appears (Home,
Grow, Devices health, Alerts current value). The LLD should define a single value
envelope in the OpenAPI spec and a single Vue component that consumes it; no screen
renders a bare number.

**Value envelope (every quantitative field the API returns):**

```
{ value, unit, provenance: measured|calculated|estimated|operator_entered,
  ts_source, age_s (server-computed at response time), stale: bool,
  staleness_max_age_s, quality: good|suspect|implausible_rejected,
  formula_version?, inputs?[], calibration_id?, sim: bool, instrument_tag? }
```

**Rule A — every number shows source age.**

1. Rendered as `value unit (age)`; age from `ts_source` against **server** `as_of` in the
   response envelope, then ticked locally. Never computed against the device clock alone
   (source clocks drift; FR-TEL-01 preserves them but the display must not trust them for
   age) and never against the browser clock relative to `ts_source` directly.
2. Age format: `< 60 s → "41 s"`, `< 60 min → "6m 40s"`, else absolute local time with
   date if not today. Tap reveals `ts_source` absolute (IST + UTC).
3. `stale: true` (backend staleness policy, FR-TEL-04) → `STALE` text tag beside the age,
   value dimmed but **still shown**. The console never decides staleness itself; the
   per-class max-age is policy the backend owns and returns (`staleness_max_age_s`), so a
   room with T/RH at 2 min and CO2 at 5 min (device-control-model §6 examples) renders
   consistently with the alert rule that fires on the same threshold.
4. Headline numbers show the **last good sample** and *its* age. A rejected sample never
   becomes a headline number; it appears in the tier/device event list and in charts as
   a hollow marker. `quality: suspect` shows a `?` tag with the reason on tap
   (rate-of-change, stuck, cross-check divergence).
5. When the client is disconnected, ages keep counting from the last successful response
   and every value is dimmed; the connection strip explains why. No cached response is
   ever restyled as fresh (ADR-0009 PWA boundary).

**Rule B — measured, calculated, estimated, simulated are never visually identical.**

| Provenance | Rendering | Phase 1 instances |
|---|---|---|
| `measured` | plain numerals, instrument tag on tap (`TE-02`, `AT-03`…) | all sensor readings; reported actuator states |
| `calculated` | `≈` prefix + `CALC` text tag; tap shows formula version and the input samples with *their* ages (the calculated value is as old as its oldest input) | VPD (FR-TEL-03) — always; DLI if the HLD ships it (see F-5) |
| `estimated` | `EST` text tag, distinct from CALC; tap shows model/revision | none expected in Phase 1 (energy/water estimates are R4). Component must support it now so R4 does not restyle |
| `operator_entered` | `ENTERED by <who>` on tap | none in Phase 1 except possibly commissioning PPFD (F-5) |
| `sim: true` | `SIM` text tag on the value, on the device, on the rack/tier, and the shell chip (§2 rule 5) | all Phase 1 virtual devices |

Tags are **text**, never colour alone — the room has glare, the operator may be
colour-blind, and a phone in a wet hand at arm's length must still read "CALC". Colour is
allowed as reinforcement only.

Acceptance (UX): an e2e test walks every screen with the sim's "stale" fault switch on,
asserts every rendered numeric has an adjacent age, asserts the VPD tile carries `CALC`
and a formula version, and asserts no numeric anywhere renders without the envelope.

---

## 5. Commissioning flow (FR-DEV-03, UC-04, J6) — step by step

Two-party by design: the **technician** (P4, `technician` role, often remote on desktop)
drives; the **on-site person** (P1, any role) does the physical steps and confirms.
Every step is an audited record with actor + timestamp (FR-DEV-03 "every step recorded",
security-architecture §5 device provisioning). The wizard is resumable: closing the
browser mid-flow leaves the device in the last recorded step, visible in the device
detail as "commissioning: step 3 of 6".

```
Devices → [+ Commission]

STEP 0  Prepare (on-site)             "Power the device on. It will appear below within
                                       ~60 s. Read the label on the board."
        data: poll GET /devices?claim_state=unclaimed

STEP 1  Claim (technician)            list of unclaimed: advertised id, hardware
                                       fingerprint, profile schema version, first seen.
                                       Technician selects one; must type the last 4
                                       characters of the fingerprint printed on the board
                                       (as read out by the on-site person) before [Claim]
                                       is enabled. Two boards announcing at once cannot be
                                       claimed by guess.
        action: POST /devices/{id}/claim {fingerprint_confirm, label}
        outcome: claim_state = claimed; device receives its per-device credential via the
                 enrolment channel (security-architecture §2 — mTLS preferred, PSK as
                 documented compromise; the UI shows which, never the secret).
        failure: fingerprint mismatch with a previously revoked/known identity →
                 quarantined, not merged; the wizard stops and shows why.

STEP 2  Discover capabilities         the device re-announces after claim. Wizard shows
        (automatic + technician       the parsed profile grouped by scope: per-tier
         verification)                instances (light.dim, airflow.*, irrigation.flood,
                                       valve.control with failsafe state), rack-level
                                       (sense.temp_rh at tier 3, sense.leak), plus any
                                       unrecognized classes ("recorded, not rendered").
                                       Technician ticks "profile matches the hardware I
                                       expect" — a human verification, not a click-through:
                                       the checklist shows the expected counts from the
                                       device type's commissioning guidance (4 fill NC,
                                       4 drain NO, 4 fans, 4 LED pairs, 1 T/RH — per
                                       instrument-tags.md) against advertised counts, and
                                       highlights every mismatch. valve.control failsafe
                                       states are cross-checked against safety-rules.json
                                       (§3.4); a mismatch is shown as a blocking fault and
                                       the wizard cannot proceed to activate.
        data: GET /devices/{id}/capabilities ; GET /safety-envelope
        action: POST /devices/{id}/commissioning/verify {accepted: true, notes}

STEP 3  Assign (technician)           pick room → rack (or room / terrace / water-skid /
                                       skid for I/O nodes). Per-tier capability instances
                                       default to tier 1..4 from the profile's instance
                                       index; technician can remap if the physical wiring
                                       differs. Zone rows are created/linked (FR-FAC-02,
                                       default 1 tier = 1 zone). A rack that already has
                                       a controller assigned shows a conflict; replacing
                                       requires retiring the old device explicitly.
        action: POST /devices/{id}/assign {rack_id, instance_map[]}

STEP 4  Identify (technician sends,   the only command in Phase 1. Semantics: a no-op /
         on-site confirms)            identify-class command that makes the controller
                                       indicate itself (status LED blink pattern on the
                                       controller board; for a virtual device, the sim
                                       harness logs and acks). It touches no valve, fan,
                                       LED bar, or pump (classification §"Safety
                                       relevance"). Wizard shows the command lifecycle
                                       live: issued → ack {SUCCESS | REJECTED reason |
                                       TIMEOUT} with latency. Then it asks the on-site
                                       person: "Did the controller on Rack 3 blink?"
                                       [Yes — I saw it] [No]. The confirming user's
                                       identity is recorded (if the on-site person is not
                                       the logged-in user, the technician records their
                                       name in a free-text field; the audit shows both).
        action: POST /devices/{id}/commands {kind: identify} → command_id;
                GET /commands/{command_id} (poll) ;
                POST /devices/{id}/commissioning/confirm-identify {seen: bool, confirmed_by}
        failure: TIMEOUT / REJECTED / "No" → wizard stays on step 4 with the ack reason
                 and a "retry" (new command id — idempotency is per id).

STEP 5  Activate (technician)         summary of all recorded steps; lifecycle →
                                       active; zone status → active (or idle with reason
                                       "no schedule" — R2 will change that wording).
        action: POST /devices/{id}/activate
        outcome: device appears in Grow under its rack with live values within one
                 staleness window; Home device-issue counts update.
```

Acceptance for this flow (release-plan: "claims a device, discovers capabilities, assigns a
tier, and a test command round-trips with ack visible"): an e2e run against a virtual
rack controller completes steps 0–5 with two distinct users (technician + operator), the
identify ack is visible with latency, the audit log contains six entries, and a second
virtual device with a mismatched failsafe advert is blocked at step 2.

---

## 6. Alert center behaviour

### 6.1 Lifecycle (FR-ALR-02) as the operator experiences it

`raised → notified → acknowledged → resolved`. Rows never vanish on their own from
Active except via `resolved`. Resolution is either rule-driven (condition cleared for its
clear-duration, backend resolves and records "auto: condition cleared") or operator-driven
with a reason. Acknowledge means "a person has seen this and is handling it"; it stops
re-notification, it does not change the row's position or severity.

### 6.2 SAFETY tier — non-dismissal (FR-ALR-03) made concrete

1. There is no "dismiss" verb anywhere in the console. For non-SAFETY alerts, "resolve
   with reason" is the closest thing; for SAFETY it is stricter.
2. A SAFETY alert is pinned first on Home, first in Alerts/Active, and in the app-shell
   banner on **every** route including Settings and the commissioning wizard. The banner
   has no close control. It is present on the Login screen only as text ("1 SAFETY alert
   active in Room 1 — sign in to view"), because the unauthenticated shell must not leak
   detail and must not suggest the alert is gone.
3. **Acknowledge requires a recorded action**: a mandatory `action_code` from a rule-
   specific list plus optional note. The server rejects an ack without one; the client
   disables the button until one is chosen. The ack is recorded in the audit log with
   actor and time (security-architecture §5).
4. **Resolve is refused while the condition is present.** The detail view shows the
   resolve control disabled with the live reason ("condition still present: 5240 ppm, 8 s
   ago"). Once the backend sees the condition cleared, the operator may resolve with a
   reason, or the rule auto-resolves — in both cases the acknowledged action stays in the
   history record.
5. **The console never claims to make the room safe.** SAFETY alert detail always
   contains the "What the hardware has done" block, sourced from the device/interlock
   event (mirror state) and the fail-safe text from `safety-rules.json`, and the sentence
   that the console mirrors, did not cause, and cannot undo the hardware action. No
   button on a SAFETY alert is labelled in a way that suggests it acts on hardware
   ("Acknowledge — record action taken", never "Clear alarm").
6. Notification channel for SAFETY (FR-ALR-04) is console + push/email regardless of
   per-severity configuration; configuration can add channels for SAFETY, not remove
   them. Where that configuration lives is flag F-4.

Phase 1 SAFETY rules are **mirrors only** (classification §"Safety relevance" point 2):
CO2 ≥ 5000 ppm (`climate.co2_safety_alarm_ppm`, verbatim), solution temperature ≥ 28 °C
(`irrigation_fertigation.solution_temperature_c.hard_inhibit_c`, verbatim), leak puck
tripped (`sense.leak` event), LSH-04 header high (event), E-stop engaged (event, if the
room I/O node reports it). Each rule row in the Rules tab shows `is_interlock_mirror: true`
and the citation. No threshold is typed into the console; the server serves it from the
rule definition that copied it from `safety-rules.json`.

### 6.3 Alert fatigue mitigation (AR-8) — what the UX does

With 1–2 operators, every unnecessary row trains them to ignore the list. Rules the LLD
must implement, all backend-side with the console displaying the result:

1. **One row per rule × scope**, never per sample. Duration counts up; the row does not
   re-raise while active. Re-notification cadence is a rule property, not a per-sample
   event.
2. **Root-cause collapse**: when a rack controller is offline, the per-tier stale-
   telemetry alerts it would generate are suppressed under the single "device offline"
   alert, which lists the affected zones. The same for room I/O node → room metrics.
   Suppressed children appear in the parent's detail ("also affects: Z-3-1..4"), not as
   rows.
3. **Duration semantics** (FR-ALR-01 threshold + duration): stale/offline rules fire only
   after the class max-age, not on the first late sample; transient broker reconnects do
   not produce rows.
4. **Severity honesty**: the sim's "implausible" fault switch produces a *device health*
   (OPERATIONAL) alert about the sensor, never a climate alert about the room — a rejected
   sample is not room news (device-control-model §6).
5. **Noisy-rule visibility**: the Rules tab shows, per rule × scope,
   `acks_without_action_30d` and `auto_resolved_under_5min_30d`. Rules above a threshold
   the HLD sets are tagged `NOISY` for review. This is the Phase 1 stand-in for the IA §5
   "flagged in maintenance review" (Maintenance screen is not in Phase 1).
6. **Per-zone thresholds** for OPERATIONAL/ADVISORY rules are configurable (FR-ALR-01),
   so a known-quirky sensor does not spam; SAFETY thresholds are not configurable in the
   console at all.
7. **No snooze.** Snooze is a dismissal with a timer; it is not offered in Phase 1 for any
   severity. If it is wanted later it goes through `/extend` with audit.

Acceptance (UX): with the sim's "offline" switch on one rack controller for 10 minutes,
the Active tab shows exactly one row for that rack (not four stale rows plus one offline
row); acking a SAFETY fixture without an action code is refused by the server and the
client; the SAFETY banner is present on every route while the fixture is active.

---

## 7. Home: "anything wrong?" in under 5 seconds

The five seconds are mostly the human's, so the design spends its budget there:

1. **One request.** `GET /facility/summary` is the only call Home needs to render the
   verdict; everything below the verdict band may load after it. NFR-PER-02 (<2 s) covers
   the request; the verdict is computed server-side so the client does no aggregation and
   cannot compute a different answer from the alerts list.
2. **The verdict is a word, not a dashboard.** `ALL CLEAR`, `n ISSUES`, `SAFETY ALERT`,
   `UNKNOWN`, in the largest type on the screen, with its `as_of` age directly under it.
   The operator reads it from across the room at arm's length before the rest paints.
3. **`ALL CLEAR` has preconditions**, all server-side: zero active alerts of any
   severity; zero devices in offline/stale/degraded/fault; room controller seen within its
   staleness window; ingest lag under its threshold; last scheduled backup succeeded. If
   any is unknown (backend cannot see the broker, room controller last-seen older than
   policy), the verdict is `UNKNOWN`, never `ALL CLEAR`. A disconnected client shows
   `UNKNOWN` with "no connection since HH:MM".
4. **Everything else on Home is one tap deep** and ordered by "what needs me": SAFETY
   rows, then active alerts, then device issues, then the room snapshot, then platform.
   No charts on Home.
5. **Polling**, not push (ADR-0009: backend-mediated only). Interval and backoff are LLD
   values; the connection strip shows the last successful `as_of` so the operator can
   judge freshness without knowing the interval.

Acceptance (UX): an e2e test loads Home cold on a throttled connection profile the LLD
defines and asserts the verdict band and its `as_of` are painted within the NFR-PER-02
budget; a second test injects a sim fault and asserts the verdict flips within one
polling interval plus the rule's duration; a third test cuts the backend and asserts the
verdict shows `UNKNOWN` and no `ALL CLEAR` text is present anywhere.

---

## 8. Carried forward from the prior frontend security review (`frontend-framework-adr/05`)

These are obligations on the scaffold/LLD, restated here so the console build cannot claim
not to have seen them:

| # | Obligation | Where it lands in Phase 1 |
|---|---|---|
| 1 | **PWA caching boundary**: service worker precaches the app shell only; all API responses (summary, alerts, telemetry, device/command status) are network-first or network-only, never cache-first/stale-while-revalidate. Now binding text in ADR-0009. | `vite-plugin-pwa` workbox config in the scaffold; e2e test that kills the backend and asserts no cached alarm/telemetry renders as current (§2 rule 4, §4 rule A5, §7 point 3) |
| 2 | **Lockfile discipline**: committed lockfile, `npm ci` in CI, dependency bumps reviewed not auto-merged; disable/audit install scripts where the toolchain allows. | `frontend/` CI job (ADR-0008 rule 4) |
| 3 | **FormKit Pro key**: if Pro is used, the key is a CI secret/env var, never a literal in `frontend/`. The commissioning wizard (§5) and alert ack forms are the first schema-driven forms; open-source FormKit core or `vue-json-schema-form` suffices for them. | LLD library pick |
| 4 | **npm scope naming**: any `@trophic-corp/*` package name is independent of the MQTT topic root; no global rename. | scaffold |
| 5 | **Re-review required** once actual `frontend/` scaffold/CI exists — the prior review pre-cleared the decision, not the code. | pipeline step 05 of this run |
| 6 | From the domain brief (03): the console **never opens an MQTT connection**; unknown capability classes are recorded and tolerated in rendering, never a crash (§3.4 "recorded, not rendered"). | LLD |
| 7 | Client-side validation is UX convenience only; the edge is the enforcing authority. Not exercised in Phase 1 (no controls) but the alert ack/resolve forms must still treat server rejection as the truth (e.g. resolve refused while condition present). | LLD |

---

## 9. Flags for the HLD (gaps, undefined workflows, boundary checks)

| Flag | Issue | Recommendation |
|---|---|---|
| **F-1** | **Facility structure authoring has no defined workflow.** Rooms/racks/tiers/zones (FR-FAC-01/02/05) must exist before commissioning step 3 can assign anything, but no Phase 1 screen creates them. IA puts "facilities/rooms" under Settings. | HLD decides: seed from a versioned config file applied at deploy (simplest, fits "no file surgery for routine ops" poorly) **or** a minimal admin-only Settings → Facility CRUD. Either way, name it; the wizard depends on it. |
| **F-2** | **`identify` is not in the capability vocabulary** (device-control-model §3). The classification requires a no-op/identify-class command for the commissioning test. | Add `control.identify` to the vocabulary (controller-scope, no actuator), or define identify as a controller-level command outside capability classes. The UX only needs: issue → ack {SUCCESS/REJECTED/TIMEOUT} + latency; on-site confirmation is human. |
| **F-3** | **Per-zone alert threshold editing** (FR-ALR-01, M) is a write surface not in the six-screen list. It is not actuation. | Keep the Rules tab read-only in v0 if the HLD wants to trim; the read endpoints (rule + citation) are required by FR-ALR-06 regardless. If edit ships, `grower`/`admin` only, SAFETY rows never editable. |
| **F-4** | **Notification channel configuration** (FR-ALR-04, S) has no home in the six screens. | Settings → Notifications (admin) or defer channel config to backend env with the console showing the effective channel per severity. SAFETY channels are additive only (§6.2 rule 6). |
| **F-5** | **DLI provenance chain.** FR-TEL-03 says DLI from light state/photoperiod, but PPFD is *not sensed in service* (absent-by-design assertion 2); any DLI needs an operator-entered PPFD from the commissioning/quarterly jig traverse. `dli_mol_m2_day` in `safety-rules.json` is TBD. | Either defer DLI to R2+ or ship it as `calculated` with an explicit `operator_entered` PPFD input shown on tap and no DLI threshold/alert. Never render a DLI number without the input chain. |
| **F-6** | **SAFETY UX cannot be exercised by the locked sim fault switches** (offline/stale/implausible produce OPERATIONAL alerts only). | Add a sim "value override" or a test-only alert fixture so §6.2 is covered by e2e; otherwise FR-ALR-03 ships untested. |
| **F-7** | **Restore/import UI vs ops runbook** (FR-BKP-02). The wizard in §3.6 is the UX if restore is in the console; the acceptance criterion only requires that a bundle restores into a clean system with verification. | HLD may keep restore as an operator CLI in v0 and show only the validation report + drill status in the console. Export-now must be in the console (J7). |
| **F-8** | **Quarantine/revoke** (security-architecture §2) has UX implications (device list state, blocked claim) but no FR-DEV ID in Phase 1. | Show `quarantined` as a claim state with reason; revoke action is admin/technician and can wait for R2 if the HLD prefers, but the list must not hide quarantined devices. |
| **F-9** | **Home "Today" rows from the IA that Phase 1 cannot fill** (batches due, zones off-target vs desired). | Omitted, not stubbed (§3.2). Record in the IA doc at docs-writer time that Phase 1 Home is the alarms + devices + connectivity subset. |
| **F-10** | **Operator/B2B mixing:** none found; no screen references customers, orders, availability. **CEA/aquarium merging:** none; the capability vocabulary is shared by ADR-0002 design, but labels are CEA (rack, tier, tray, flood) and the console must not introduce generic "tank"/"unit" wording to keep it "reusable". | Reviewer check at LLD. |
| **F-11** | **Rack position map** (IA "drill by map or list"): FR-FAC-05 gives position as a reference string only. | List view only in Phase 1; position string shown on the rack row. No map component. |

---

## 10. Consolidated UX acceptance criteria (for the LLD and e2e)

1. Every rendered numeric on every screen carries age + provenance from the value
   envelope; no bare numbers (§4).
2. `ALL CLEAR` is never shown when any input is unknown or the client is disconnected;
   `UNKNOWN` is (§7).
3. SAFETY banner on every authenticated route while a SAFETY alert is active; no close
   control; ack requires an action code (server + client); resolve refused while the
   condition persists (§6.2).
4. Root-cause collapse: one row for an offline rack controller, not one per tier (§6.3).
5. Commissioning completes steps 0–5 against a virtual device with two users; identify
   ack and latency visible; failsafe mismatch blocks at step 2 (§5).
6. No metric row renders for a capability no device at that scope advertises: no per-tier
   CO2, per-tray level, fixed PPFD, or per-tier T/RH anywhere (§3.3).
7. No "Desired" column, no setpoint, no control affordance renders anywhere in Phase 1
   (§3.3); the only command surface is `identify` on device detail / wizard.
8. Virtual devices carry `SIM` at value, device, rack/tier, and shell level (§2 rule 5).
9. Unbuilt IA sections are absent from navigation, not greyed (§2).
10. App shell loads with the backend unreachable and states so; no cached API response
    renders as current (§8 item 1).
11. Primary actions are single-tap, large-target, no hover-only or drag-only paths; the
    phone layout is the primary layout for Home, Grow, Alerts; desktop is primary for the
    commissioning wizard and Settings (§1).
12. All timestamps IST with UTC on tap (§2 rule 7).

---

## Appendix — read endpoints implied by the wireframes (for the OpenAPI draft)

```
GET  /session                         GET  /facility/summary
GET  /facility/tree                   GET  /rooms/{id}/snapshot
GET  /racks/{id}/snapshot             GET  /zones/{id}/snapshot
GET  /zones/{id}/telemetry            GET  /zones/{id}/events
GET  /devices                         GET  /devices/{id}
GET  /devices/{id}/capabilities       GET  /devices/{id}/commands
GET  /devices/{id}/events             GET  /commands/{id}
GET  /alerts                          GET  /alerts/{id}
GET  /alert-rules                     GET  /alert-rules/{id}
GET  /users                           GET  /backups
GET  /backups/jobs/{id}               GET  /backups/{id}/download
GET  /system/health                   GET  /safety-envelope

Writes (non-actuation): POST /auth/login|logout · POST /devices/{id}/claim|assign|activate
· POST /devices/{id}/commissioning/verify|confirm-identify · POST /devices/{id}/commands
(identify only) · POST /alerts/{id}/ack|resolve · PATCH /alert-rules/{id} (F-3)
· POST/PATCH /users · POST /users/{id}/totp/reset · POST /backups/export
· POST /backups/import/validate · POST /backups/import (F-7)
```

Every endpoint is org-scoped by session (ADR-0005); the console never sends an org id.
