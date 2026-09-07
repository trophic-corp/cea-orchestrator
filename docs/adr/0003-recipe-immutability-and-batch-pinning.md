# 0003. Immutable recipe versions, pinned by batches

**Status:** proposed
**Date:** 2026-09-07
**Pipeline artifact:** docs/product/2026-09-07-workspace-audit.md (foundation analysis)

## Context

Working rule 15: "Never modify historical recipe definitions silently." The platform's
R&D value depends on answering, years later, "which exact configuration produced this
result?" (rd-data-model §4). Recipes are also reusable templates operators will clone,
tune, and re-save (UC-08), and they carry safety-envelope-constrained parameters
(safety-rules.json bounds — e.g., PPFD operating band, photoperiod bounds, flood depth/
dwell) validated at both save and apply time (FR-RCP-04).

## Decision

1. `Recipe` is a named container; all content lives in `RecipeVersion`s that are
   **immutable once any ProductionBatch references them** — enforced at the database
   level (policy/trigger), not by application discipline alone.
2. Batch → RecipeVersion references store the **version id plus a content hash**; the
   hash is verified on read, so tampering or accidental mutation is detectable.
3. All changes produce a new version with `cloned_from` lineage — recipe history is a
   tree; comparisons (R6 analytics) walk the tree.
4. Version labels: `MAJOR.MINOR` — MAJOR for changes to parameters a running batch
   depends on (lifecycle/safety-relevant), MINOR for advisory metadata. Labels are
   human-facing; ids and hashes are the machine truth.
5. Templates are simply versions flagged `is_template`; applying and modifying an
   instance never writes back to the source (save-as workflow, FR-RCP-03).
6. Safety-envelope validation happens at **save** (warning + block on hard violations)
   and at **apply** (hard block), using bounds sourced from `safety-rules.json`; the
   envelope itself is not part of recipe content and can tighten without invalidating
   stored versions (historical batches keep their hash-verified content; they simply
   cannot be re-applied under newer, tighter envelopes — and that fact is recorded).

## Consequences

- Historical batches are permanently explainable; no audit archaeology needed.
- Operators must be trained into "edit = new version" (UX burden — mitigated by the
  save-as flow in the recipe editor, J-recipes workflow).
- Storage grows with version count (negligible; JSONB rows).
- A badly-parameterized published recipe stays in history (correct — it's data);
  `retired` status marks "don't offer this in the picker" without deleting anything.
- Re-applying an old version under a newer safety envelope may legitimately fail —
  surfaced clearly at apply time with the conflicting bound cited.

## Alternatives considered

- **Mutable recipes with audit-log diffs**: reconstructing "what was the config on
  Sept 10" becomes log forensics; hash-verifiable pinning is impossible; rejected.
- **Copy-on-write per batch (each batch gets a private snapshot)**: preserves
  immutability but loses the template lineage/comparison story and duplicates content
  blindly; the lineage tree + content hash achieves both without duplication.
- **Snapshot-only (no versions)**: same effect as copy-on-write with worse ergonomics;
  rejected.