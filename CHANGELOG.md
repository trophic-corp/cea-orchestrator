# Changelog

All notable changes to this project are documented here. Format loosely follows
[Keep a Changelog](https://keepachangelog.com/); versioning follows semver once the first
release cuts. `release-manager` owns this file.

## [Unreleased]

### Added

- Initial workspace bootstrap: repository layout, 21 specialist agents, `/ship` `/extend`
  `/fix` `/health-check` pipeline commands, ingested domain knowledge base
  (`knowledge/cea/`), sourced physical safety thresholds
  (`.github/agentic-rules/safety-rules.json`), Phase 1 CI/CD scaffold.
- **ADR-0006** — Go as the backend implementation language (resolves OQ-16).
- OQ-3 room-controller-platform decision brief (`docs/pipeline/oq3-room-controller-platform/`).
- OQ-16 spike protocol and scorecard (`docs/pipeline/oq16-backend-language-spike/`),
  retained as the record of how the collapsed decision would have been scored.
- `.gitattributes` enforcing LF, plus DrvFs-appropriate git stat settings.

### Changed

- **Phase 1 kickoff (2026-09-08).** ADR-0001…0005 approved as written and moved
  `proposed` → `accepted`. Phase 1 (Release 1) scope locked; Releases 2–8 remain proposed.
- **OQ-1 closed: saffron deferred indefinitely.** README, CLAUDE.md, vision, and the
  domain model updated to microgreens-first. Saffron material in `knowledge/cea/` and
  `safety-rules.json` retained as reference data, not active scope.
- Model/provider migrated off GLM-5.3/OpenRouter to Claude via subscription OAuth;
  routing config removed from `.claude/settings.json`. Closes risk AR-5 — visual
  verification of the scanned engineering drawings no longer needs a separate session.
- `system-architecture.md` §2 backend language Node.js → Go, per ADR-0006.
- Repository normalized to LF line endings, removing 73 files of phantom diff.

### Notes

- Phase 1 build is **paused pending OQ-3** (room controller compute platform) at the
  owner's direction — decision brief ready, closes as ADR-0007.
- `gh` is unauthenticated on this host: `github-ops-agent` cannot open PRs or verify
  branch protection until `gh auth login` is run.
