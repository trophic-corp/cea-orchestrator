# 0008. Monorepo for now, with named split triggers

**Status:** accepted
**Date:** 2026-09-08 — owner decision at Phase 1 kickoff
**Pipeline artifact:** none — decided in conversation at kickoff, not through a `/ship` run
**Supersedes:** the unexamined monorepo assumption in the workspace audit §3.2 item 10

## Context

The repository has `backend/`, `frontend/`, `firmware/`, and `hardware/` directories, all
containing only `.gitkeep` — **no code exists anywhere in the repo yet**. They are wired
into CI (a `detect` job probing for per-subsystem manifests) and into
`docker-compose.prod.yml` (`build: ./backend`, `build: ./frontend` — local build contexts,
i.e. monorepo assumptions).

The owner asked whether these were documentation-only, and raised a preference for separate
repositories per service/firmware, with the goals: **modular, maintainable, easy to extend**.

Investigating provenance showed the monorepo was **never a weighed decision**. It appears
once, in workspace-audit §3.2 item 10, under *"Platform/infra (scaffolding + research)"* —
the same category, and the adjacent numbered item, to the Node.js assumption that the audit
itself flagged as *"scaffold assumption, not a weighed decision"* and that ADR-0006
overturned. No ADR covered repository structure. The docs also contradicted themselves,
CLAUDE.md instructing agents to create **repos** (plural) under an org while the layout was
a single repo.

Three findings shaped the decision:

1. **Repo boundaries are not module boundaries.** The stated goals are bought with
   interface discipline, not repo count. The architecture already commits to this:
   *"backend (Go, single modular service — boundaries as modules, not microservices)."* A
   polyrepo without interface discipline is a distributed ball of mud — worse than a
   co-located one, because the coupling becomes invisible and unenforceable across repo
   lines. Splitting can hide poor modularity as easily as it can govern good modularity.

2. **Reversibility is strongly asymmetric.** Monorepo → split is a scripted,
   history-preserving directory extraction (`git subtree split` / `filter-repo`), cheap
   whenever performed. Split → monorepo means reconciling divergent CI, versioning, and
   accumulated contract drift, plus unwinding team coordination habits. The monorepo is the
   reversible default.

3. **The input that determines the boundary is not yet known.** [OQ-3] decides what
   "firmware" even means. If the room controller is a Linux SBC, there are three kinds of
   code — ESP32 rack firmware (C/ESP-IDF), room controller service (Go), backend (Go) — and
   the natural boundary is **MCU vs Linux**, not "firmware vs backend". If it is an MCU,
   firmware is one thing. OQ-3 is the question the Phase 1 build is already paused behind.

The strongest argument *for* splitting immediately — access control, the one thing a repo
boundary provides that a module cannot — **does not survive the evidence**. Phase E names
fabrication as permanently outsourced to Coimbatore vendors but embedded/firmware
engineering as **never-outsource IP** (*"durable capability, not vendor-dependent"*). The
outsourced vendors need manufacturing packs — Gerbers, BOMs, drawings — not repository
access. There is currently no party who needs partial access.

## Decision

**Stay a monorepo for now.** Do not split `backend/`, `frontend/`, `firmware/`, or
`hardware/` into separate repositories at this time.

This is recorded as a decision with a **review condition**, not as deferral. Any of the
following triggers a revisit — and the revisit is a `/ship`-worthy decision producing a
superseding ADR, not an ad-hoc move:

| # | Trigger | Why it changes the answer |
|---|---|---|
| T1 | **[OQ-3] lands on a Linux SBC or IPC** | The boundary shape changes from firmware-vs-backend to MCU-vs-Linux |
| T2 | **Aquascaping/aquarium platform work begins** | `vision.md` and the IA both call it a *separate product with shared platform capabilities* — that is when shared libraries need their own home. **The most likely real trigger.** |
| T3 | **Product firmware ships on customer hardware** (Phase E Year 2: climate controller 2.1, fertigation controller 2.2) | Product firmware is not facility firmware — different audience, release cadence, and support obligation |
| T4 | **Anyone outside the core team needs partial access** | Access control is the one thing modules cannot provide |
| T5 | **KiCad/STEP binaries make clone times painful** | Large binaries are permanent in history; extract before it hurts |
| T6 | **Firmware OTA cadence genuinely decouples from backend deploys** | Independent release trains want independent repos |

## Consequences

**Easier**
- Cross-cutting changes stay atomic. A capability-model change (ADR-0002) touching firmware
  capability advert + backend registry + frontend display is one commit and one PR, not
  three coordinated PRs across three repos — which matters disproportionately at bus
  factor 1 (AR-13).
- `/ship` and `/extend` keep working as designed: `docs/pipeline/<slug>/` artifacts and
  `docs/adr/` stay adjacent to the code they govern, so `github-ops-agent` can link them by
  relative path and `/extend`'s ADR re-use needs no cross-repo lookup.
- Phase 1 CI/CD stays as built: one compose stack with local build contexts, no image
  registry or cross-repo version pinning required yet.

**Harder**
- Access control is all-or-nothing. Granting anyone access grants everything (T4).
- Nothing structurally *prevents* cross-subsystem coupling — only the discipline below
  does. This is the real cost of the decision and it must be actively held.

**Discipline that keeps extraction cheap — binding, and mirrored into CLAUDE.md so agents
enforce it at build time rather than reviewers catching it later:**

1. The **OpenAPI spec is the contract of record** between backend and frontend (ADR-0006).
   Never share types across subsystem lines by file path — generate them.
2. Firmware ↔ backend couple **only** through the versioned MQTT topic/payload schema
   (device-control-model §4). No shared source.
3. **No cross-directory imports** between `backend/`, `frontend/`, and `firmware/`.
4. Each subsystem keeps its **own build and dependency manifest**, and its CI job stays
   independently runnable.

Hold these and a later split is a directory move. Fail to hold them and splitting would not
have helped anyway — the coupling would simply have become cross-repo coupling, which is
harder to see and harder to fix.

## Alternatives considered

**Full split by subsystem now** (backend/frontend/firmware/hardware as separate repos, this
repo becoming docs + orchestration). Cleanest boundaries and best access control. Rejected
for now: it makes every cross-cutting change a multi-PR coordination problem at bus
factor 1; it requires designing a cross-repo convention for ADR and pipeline-artifact
linking before the orchestration model still works; and it would commit to a boundary shape
before [OQ-3] reveals what that shape should be. **Reconsider at T1/T2.**

**Split firmware + hardware only, keeping backend + frontend + docs together.** The
strongest of the split options — firmware has a genuinely separate toolchain and release
cadence, and KiCad binaries do bloat clones. Rejected now only on timing: its main
justification (contractor access) is void per Phase E's never-outsource-firmware position,
and T1 may redefine what belongs in a firmware repo within weeks. **This is the most likely
shape when a trigger fires.**

**Keep the monorepo silently, as-is.** Rejected. That is how the Node.js assumption survived
unexamined until OQ-16; leaving repo structure in the same state invites the same
re-litigation. The point of this ADR is that the monorepo is now *chosen* rather than
*inherited*, with the conditions for changing it written down.
