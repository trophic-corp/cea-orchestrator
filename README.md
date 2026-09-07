# trophic

Controlled Environment Agriculture (CEA) + aquascaping hardware R&D program — saffron
cultivation, hydroponic/aquascaping product lines. R&D room in Ooty, manufacturing base in
Coimbatore, Tamil Nadu, India.

This repository is a multi-agent Claude Code workspace: requirement → domain review →
architecture → security review → implementation → deployment, carried coherently across
every feature via Architecture Decision Records rather than re-litigated per change. See
`CLAUDE.md` for the full model, and `docs/pipeline/README.md` for why the orchestration is
shaped the way it is.

## Start here

- `knowledge/cea/` — literature review, global benchmark, product portfolio, BOM/supplier
  catalog, roadmap, and OCR'd rack/room/water-recovery engineering references.
- `.github/agentic-rules/safety-rules.json` — sourced physical/process safety thresholds.
  Never invent a number that isn't here or in `knowledge/cea/`.
- `docs/adr/` — accepted architectural decisions.
- `.claude/agents/` — the 21 specialist agents this workspace runs on.
- `.claude/commands/` — `/ship`, `/extend`, `/fix`, `/health-check`.

## Repository layout

```
knowledge/cea/          literature review, benchmark, portfolio, BOM, roadmap + OCR'd CAD refs
docs/adr/                Architecture Decision Records
docs/pipeline/            per-requirement working artifacts
docs/site/                Starlight documentation site
firmware/                 ESP32 / embedded
backend/                  services
frontend/                 dashboards / UI
hardware/                 PCB/KiCad refs, pinout maps
.github/agentic-rules/safety-rules.json
.claude/agents/           specialist agent definitions
.claude/commands/         /ship, /extend, /fix, /health-check
docker-compose.prod.yml   Phase 1 prod stack
```

## Model / provider

This workspace runs on GLM-5.3 via `.claude/settings.json` — see `CLAUDE.md` for the
provider config and its text-only limitation (relevant to the scanned PDFs in
`knowledge/cea/cad/`).
