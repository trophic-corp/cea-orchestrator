---
description: Full pipeline for a new capability with no existing ADR coverage
argument-hint: [requirement description]
---
Requirement: $ARGUMENTS

Create `docs/pipeline/<slug>/` (derive `<slug>` from the requirement — short, kebab-case).
Gate every stage on the previous artifact existing and not marked BLOCKED. If a gate blocks,
stop and report to the user rather than proceeding past it.

1. @architecture-reviewer → `01-classification.md` (kind, change_class; if this is actually
   incremental, say so and stop — recommend `/extend` instead)
2. Fan out in parallel to the Product & Domain Council members relevant to `kind`
   (@botany-horticulture-specialist, @electronics-hardware-specialist,
   @mechanical-industrial-design-specialist, @manufacturing-supply-chain-specialist,
   @product-strategy-specialist — not all five are relevant to every requirement, use
   judgment on which apply) → `02-domain-notes/*.md`
3. @systems-integration-reviewer → `03-domain-brief.md` (reconciled; asks the user if
   specialists disagree and it can't resolve it)
4. @hld-architect → `04-hld.md` (+ draft ADR in `docs/adr/` if this is a lasting decision)
5. @security-safety-reviewer → `05-security-review.md` — BLOCK gate
6. @lld-architect → `06-lld.md`
7. @systems-integration-reviewer → re-check LLD against the domain brief (cheap
   re-verification, appends to `03-domain-brief.md`)
8. The implementation specialist(s) selected by `kind` in step 1
   (@firmware-engineer / @backend-engineer / @frontend-engineer / @pcb-layout-engineer) →
   code + unit tests + `08-implementation-notes.md`
9. @build-deploy-engineer → `09-build-report.md`
10. @automation-engineer → `10-automation-notes.md` (hardware-in-the-loop pass required if
    kind is firmware/hardware)
11. @qa-e2e-validator → `11-e2e-report.md` — BLOCK gate on PRD acceptance criteria
12. @release-manager → version bump + CHANGELOG entry
13. @github-ops-agent → opens the PR (never merges), links every artifact above
14. @docs-writer → Starlight page in the same PR; flips the ADR to `accepted` once merged

For `kind: docs-only`, skip steps 2–7 and 10. Finish with a one-paragraph summary and the PR
link. Never merge, force-push, or change branch protection without the user's explicit
confirmation.
