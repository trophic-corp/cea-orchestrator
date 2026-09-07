# Pipeline artifacts

Every `/ship`, `/extend`, and `/fix` invocation writes its working artifacts to
`docs/pipeline/<slug>/` — classification, domain notes, the reconciled domain brief,
HLD/LLD, security review, implementation notes, build/automation/e2e reports. These are the
paper trail `github-ops-agent` links into every PR, so a reviewer (human or agent) can trace
why a decision was made without re-deriving it.

`docs/pipeline/health-reports/` holds `maintainer-agent`'s periodic, non-blocking output from
`/health-check` — one dated report per run.

## Why this workspace has three pipeline commands instead of one

A single linear "requirement → ship" pipeline doesn't fit a program that has to carry
decisions coherently across a long-running hardware + software system:

- **Uniform review cost is wrong.** A new saffron-corm vernalization feature and a one-line
  dashboard copy fix don't carry the same risk, and paying full Product Council + HLD +
  security review for both wastes the council's time on the trivial change and — worse —
  trains everyone to route around the process for anything that feels like it shouldn't need
  it.
- **Domain knowledge has to be upstream of code, not a review-time surprise.** If a
  botanist's tolerance and a sensor's actual precision disagree, that needs catching before
  the HLD is written (`systems-integration-reviewer`'s job), not discovered in a PR review
  after the firmware is already built around the wrong assumption.
- **Maintainability decays if it's a pipeline stage instead of a habit.** A tech-debt
  checklist item at the end of every feature pipeline gets skipped under deadline pressure,
  every time, by every team. `/health-check` runs on its own clock, independent of any
  single requirement, specifically so it can't be skipped by scope pressure on a feature.

So: `/ship` for greenfield (full Product Council → cross-reference → HLD/ADR → security
review → LLD → build → ship), `/extend` for a successive feature on an already-decided
subsystem (only the touched specialist(s), a lightweight consistency check instead of a
fresh HLD, security review still never skipped), `/fix` for maintenance (no fresh design
unless the fix reveals the design itself is wrong), and `/health-check` running independently
of all three, non-blocking, issue-only.

ADRs in `docs/adr/` are what make `/extend`'s lighter path *safe* rather than just faster —
it's re-using a decision that was actually reasoned through in a prior `/ship`, not skipping
the reasoning.

## CI/CD phasing

**Phase 1 (built now):** one prod stack — `docker-compose.prod.yml` +
`.github/workflows/build-test-deploy.yml`. Push to `main` builds, tests, and deploys. Merges
are gated on this plus required review.

**Phase 2 (not built yet — build only when explicitly asked):** an integration stack,
`docker-compose.integration.yml`, spun up per-PR, with e2e run against it instead of prod;
promote to prod only after it passes. This is the natural next step once the Phase 1 stack
has real services in it and PR volume justifies the extra infrastructure, but building it
before there's anything to test against it would be premature.
