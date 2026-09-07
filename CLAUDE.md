# trophic — CEA + aquascaping hardware R&D workspace

Controlled Environment Agriculture (saffron cultivation, hydroponic/aquascaping product
lines) R&D program, manufacturing base Coimbatore, R&D room Ooty, Tamil Nadu, India.

This is a multi-agent, multi-phase workspace: requirement → domain review → architecture →
security review → implementation → deploy, carried coherently across every feature via
Architecture Decision Records, not re-litigated per change. See `docs/pipeline/README.md`
for the full reasoning behind the orchestration model.

## Before doing anything domain-specific

Read what's relevant first:

- `knowledge/cea/README.md` — index of the literature review, benchmark, product portfolio,
  BOM, and roadmap documents, plus an honest account of which scanned-PDF pages OCR'd
  usably and which need a visual (non-GLM) pass.
- `docs/adr/` — accepted architectural decisions. Check here before proposing a new HLD for
  something that may already have one; `/extend` assumes this is current.
- `.github/agentic-rules/safety-rules.json` — sourced physical/process safety thresholds
  (VPD, CO2, pH/EC gates, valve fail-safe states, RCBO requirements, flood/leak interlock
  timing). **Never invent a threshold.** If a value isn't in this file and isn't in
  `knowledge/cea/`, it's genuinely unknown — stop and ask a human rather than guess. Several
  entries are explicitly marked `TBD — needs domain expert input`; treat those as blocking
  for anything that would ship a physical control loop around that value.

## Model / provider

This workspace runs on GLM-5.3 (Zhipu/Z.ai), routed through OpenRouter's Anthropic-compatible
endpoint, via `.claude/settings.json` — see that file for the endpoint config. `$OPENROUTER_API_KEY`
must be exported in the shell environment before starting a session; it is never committed.

**GLM-5.3 is text-only — it cannot read images/PDFs directly.** Three of the source
documents are scanned PDFs with no text layer; they've been OCR'd into `knowledge/cea/cad/`
as `.ocr.md` companions. OCR quality is documented per-page in
`knowledge/cea/README.md` — some pages (diagrams, dimension drawings, one specific EC gate
value) are flagged as unreliable and need a real Claude session (run `claude-anthropic`, not
plain `claude`, which is GLM-backed in this repo) for a visual read before being trusted.

## Orchestration model

Three entry points sized to the change, plus one always-on safety gate and one periodic
maintenance loop:

- **`/ship`** — new capability, no existing ADR coverage. Full Product & Domain Council →
  cross-reference → HLD (+ new ADR) → security review → LLD → build → ship.
- **`/extend`** — a feature on a subsystem that already has an ADR. Only the specific
  domain specialist(s) touched are re-consulted; lightweight consistency check instead of a
  fresh HLD; security review still never skipped.
- **`/fix`** — bugfix/maintenance. No fresh HLD/LLD unless the fix reveals an actual design
  flaw, in which case it escalates to `/ship` or `/extend` instead of patching around it.
- **`/health-check`** — periodic, non-blocking. Surfaces problems as GitHub issues only;
  never opens a PR, never edits code.

Specialist agents are defined in `.claude/agents/`; every one of them is instructed to
consult `knowledge/cea/`, `docs/adr/`, and `safety-rules.json` before producing output, and
to write its artifact to the `docs/pipeline/<slug>/` path it's given.

## Guardrails (see `.claude/settings.json` for the enforced allow/ask list)

- `git push --force`, `git reset --hard`, branch deletion, anything touching
  `.github/workflows/*.lock.yml`, `gh secret set`, and `gh api PATCH/DELETE` on repo
  settings all require explicit confirmation.
- `security-safety-reviewer`, `qa-e2e-validator`, and `maintainer-agent` have no `Edit` tool
  — review/report only.
- Any agent hitting genuine ambiguity in a physical safety threshold stops and asks rather
  than guessing.
- Repos are created under the `trophic` GitHub org (`gh repo create trophic/<name>`);
  `github-ops-agent` is the only agent that creates repos, opens PRs, sets branch
  protection, or manages secrets, and it never merges automatically.

## Repository layout

```
knowledge/cea/        literature review, benchmark, portfolio, BOM, roadmap + OCR'd CAD refs
docs/adr/              Architecture Decision Records — check before designing anything new
docs/pipeline/         per-requirement working artifacts (PRD, HLD, LLD, reviews)
docs/pipeline/health-reports/   periodic maintainer-agent output
docs/site/             Starlight documentation site
firmware/               ESP32 / embedded
backend/                services
frontend/               dashboards / UI
hardware/               PCB/KiCad refs, pinout maps
.github/agentic-rules/safety-rules.json   sourced physical/process safety thresholds
.claude/agents/         specialist agent definitions
.claude/commands/       /ship, /extend, /fix, /health-check
```

## CI/CD

Phase 1 (current): one prod stack, `docker-compose.prod.yml` + a single GitHub Actions
workflow (build → test → deploy on push to `main`). Phase 2 (a per-PR integration stack,
`docker-compose.integration.yml`) is intentionally not built yet — see
`docs/pipeline/README.md`.
