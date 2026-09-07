# Operations

**Owner:** `build-deploy-engineer` (deploy), `maintainer-agent` (periodic health,
issues only). Populated as the stack becomes real (Release 1 deploy).

Content to live here when written:

- `runbook.md` — startup/shutdown, restore-from-backup drill, broker/DB health,
  certificate/credential rotation, alert-channel config.
- `backup-restore.md` — the operator-facing procedure matching FR-BKP-01..03.
- `degradation.md` — what the operator sees and does when: internet down, host down,
  edge controller down, sensor faulted, E-stop engaged (mirrors the reliability matrix
  in docs/architecture/system-architecture.md).
- `health-reports.md` → links to `docs/pipeline/health-reports/` (maintainer output).

Physical infrastructure notes (power, RCBO, E-stop, terrace tank) live with the
hardware docs + safety-rules.json — this directory covers the software stack only.