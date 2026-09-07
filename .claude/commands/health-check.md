---
description: Run the maintainer-agent health scan and file issues for anything found
---
@maintainer-agent scans the repo for stale dependencies, coverage drift, flaky/skipped tests,
ADRs that no longer match the code, docs referencing removed features, and drift in
`.github/agentic-rules/safety-rules.json`. Write `docs/pipeline/health-reports/<date>.md`,
and open one GitHub issue per distinct finding (never a PR, never an edit). Report a summary
to the user, with any finding touching a safety-relevant subsystem called out first.
