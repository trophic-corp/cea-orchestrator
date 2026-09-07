# Architecture Decision Records

This directory is the backbone of the orchestration model described in
`docs/pipeline/README.md`: it's what lets `/extend` re-use a prior decision instead of
re-deriving it, and what `architecture-reviewer` checks to classify a requirement as
greenfield vs. incremental.

## Numbering and status

- Files are named `NNNN-short-title.md`, sequential, zero-padded to 4 digits
  (`0001-...`, `0002-...`).
- Status is one of `proposed` (drafted by `hld-architect`, not yet merged),
  `accepted` (its PR has merged — only `docs-writer` flips this), `superseded` (a later ADR
  replaces it — link both ways), or `deprecated`.

## Template

Use `0000-template.md` as the starting structure for a new ADR: Context, Decision,
Consequences, Alternatives Considered.

## Who touches this directory

- `hld-architect` drafts new ADRs (status `proposed`) when a design decision will outlive
  the single requirement it's part of.
- `docs-writer` flips status to `accepted` once the ADR's PR has actually merged — never
  before.
- `architecture-reviewer`, `systems-integration-reviewer`, and every domain specialist read
  this directory before producing output, per each agent's instructions.
- `maintainer-agent` periodically spot-checks whether an ADR still matches what the code
  actually does, and files an issue (never edits) if it's drifted.
