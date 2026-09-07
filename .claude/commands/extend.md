---
description: Lighter pipeline for a feature that extends a subsystem with existing ADR coverage
argument-hint: [requirement description]
---
Requirement: $ARGUMENTS

Create `docs/pipeline/<slug>/` (derive `<slug>` from the requirement).

1. @architecture-reviewer confirms `change_class: incremental` against `docs/adr/` and
   identifies which existing HLD/ADR this extends. If it's actually greenfield, stop and
   recommend `/ship`.
2. Consult only the specific Product & Domain Council specialist(s) whose domain the
   increment actually touches — not the full council. Write to `02-domain-notes/*.md` as in
   `/ship`, but only for the specialist(s) invoked.
3. @systems-integration-reviewer does a lightweight check of the increment against the
   existing ADR/domain brief it's extending (not a fresh reconciliation) → appends to or
   creates a scoped `03-domain-brief.md`.
4. @lld-architect updates the existing LLD (identified in step 1) rather than starting from
   scratch → `06-lld.md`.
5. @security-safety-reviewer — never skipped → `05-security-review.md`, BLOCK gate.
6. The implementation specialist(s) selected by `kind` → code + unit tests +
   `08-implementation-notes.md`.
7. @build-deploy-engineer → `09-build-report.md`.
8. @automation-engineer → `10-automation-notes.md` (HIL pass required if kind is
   firmware/hardware).
9. @qa-e2e-validator → `11-e2e-report.md` — BLOCK gate.
10. @release-manager → version bump + CHANGELOG entry.
11. @github-ops-agent → opens the PR (never merges), links every artifact above and the ADR
    it extends.
12. @docs-writer → updates the relevant Starlight page in the same PR.

Never merge, force-push, or change branch protection without the user's explicit
confirmation.
