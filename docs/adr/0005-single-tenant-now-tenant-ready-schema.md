# 0005. Single-tenant now, tenant-ready schema

**Status:** accepted
**Date:** 2026-09-07 · **Accepted:** 2026-09-08 — owner approval at Phase 1 handover, approved as written
**Pipeline artifact:** docs/product/2026-09-07-workspace-audit.md (foundation analysis)

## Context

Trophic may eventually sell hardware + software to other farms (persona P6); the brief
asks for a Tenant → Organization → Facility → Room → Rack → Tier capability "eventually"
while explicitly warning against premature SaaS complexity (working rules 8, 9). Today
there is exactly one operator org (Sholaverde), one facility, and no external billing,
signup, or fleet-admin requirements.

## Decision

1. **Deploy single-tenant.** No tenant signup, no org admin UI, no cross-tenant
   anything. One organization record exists in the database.
2. **Design tenant-ready:** every core table carries `org_id`; the data-access layer is
   org-scoped from the first query written (no `SELECT` without org filter, enforced by
   a query-layer convention + tests); MQTT topic taxonomy includes org/site segments
   (ADR-0004) so multi-tenant brokers need no re-layout; authz roles are defined
   org-relative.
3. **Do not** build: org switching UI, tenant-aware billing, per-tenant encryption keys,
   data residency, or a fleet-management console. These activate at productization time
   (Release-independent decision; revisit when the first external customer is real).

## Consequences

- Productization later is an auth/routing/deployment problem — not a data migration.
- Slight cost now: org_id plumbing and query discipline from day one (cheap; retrofitting
  is the expensive path).
- Risk to watch: query-layer discipline can erode; automated tests asserting org-scoping
  on every repository method are part of the acceptance criteria of the first backend
  /ship (roadmap R1).
- A second facility for the *same* org (Sholaverde scaling) works with zero changes —
  facility rows, not tenants.

## Alternatives considered

- **Full multi-tenancy now (org model, tenant admin, invitation flows)**: months of work
  serving zero current users; violates working rule 8; rejected.
- **Pretend tenancy doesn't exist (no org_id, retrofit later)**: retrofit means touching
  every table, every query, and every MQTT topic with live production data — the exact
  migration the working rules warn about; rejected.
- **Row-level security (RLS) enforced in the DB now**: adds complexity to every
  interaction for a single-tenant deployment; the query-layer convention + tests is
  sufficient discipline until a second real tenant exists; revisit RLS at productization.