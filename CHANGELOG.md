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
- **ADR-0008** — monorepo for now, with six named triggers (T1–T6) that reopen the split
  decision. Converts an unexamined scaffold assumption into a real decision with a review
  condition. Boundary discipline mirrored into CLAUDE.md so agents enforce it at build time.
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
- GitHub org corrected `trophic` → `trophic-corp` in CLAUDE.md, `github-ops-agent`, and the
  audit, matching the actual remote. Recorded as a **placeholder expected to change before
  release**, with a warning against renaming it by find-replace: the MQTT topic root
  `trophic/<org>/<site>/…` (ADR-0004) shares the word and does not track the GitHub org.
- CI `detect` job now probes `backend/go.mod` and the backend job builds with Go, replacing
  the `package.json`/npm assumptions ADR-0006 superseded. Left unfixed, the backend job
  would have silently never run.

### Notes

- Phase 1 build is **paused pending OQ-3** (room controller compute platform) at the
  owner's direction — decision brief ready, closes as ADR-0007.
- `gh` is unauthenticated on this host: `github-ops-agent` cannot open PRs or verify
  branch protection until `gh auth login` is run.
