---
description: Root-cause a bug, fix it, regression-test it — no fresh design unless the fix reveals one is needed
argument-hint: [bug description or issue link]
---
Issue: $ARGUMENTS

Create `docs/pipeline/<slug>/` (derive `<slug>` from the issue).

1. @architecture-reviewer identifies the affected subsystem, `kind`, and whether it's
   safety-relevant per `.github/agentic-rules/safety-rules.json` → `01-classification.md`.
   If the root cause is actually a design flaw (the existing ADR/HLD is wrong, not just the
   implementation), stop and recommend `/ship` or `/extend` instead of patching around it.
2. The relevant implementation specialist (@firmware-engineer / @backend-engineer /
   @frontend-engineer / @pcb-layout-engineer, per `kind`) fixes it and writes a regression
   test → `08-implementation-notes.md`.
3. @security-safety-reviewer → `05-security-review.md` — only if step 1 flagged this as
   safety-relevant. BLOCK gate when invoked.
4. @qa-e2e-validator → `11-e2e-report.md`, scoped to the affected area only — BLOCK gate.
5. @release-manager → patch version bump + CHANGELOG entry.
6. @github-ops-agent opens the PR (never merges), links the artifacts above.

Never merge, force-push, or change branch protection without the user's explicit
confirmation.
