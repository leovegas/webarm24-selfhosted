# Changelog — WebArm24 Self-hosted

Each entry lists the image **build date**. Your license runs every image built on or before its `updates_until` ([04](04-license.md)). The entries also say whether `.env` changes or migrations are needed.

<!-- vendor: newest first. Template:
## X.Y.Z — YYYY-MM-DD (build date)
**Upgrade notes:** none | add `NEW_VAR=…` to .env | …
### New
### Fixed
### Security
-->

## 1.0.0 — first release
**Upgrade notes:** first release.
### New
- Self-hosted edition of WebArm24:
  - pipelines, workflows, browser agents, `ai_agent` steps with skills and MCP servers;
  - the MCP endpoint for coding agents;
  - sandboxed Python steps with filtered egress and approved packages;
  - version history, data storage, schedules and webhooks.
- Offline Ed25519 license keys with seat limits and an update window. Optional weekly check-in for automatic key renewal.
- `webarmctl`: preflight, install, status, doctor, license, upgrade (drain + automatic rollback), rollback, backup, restore, smoke, support-bundle, all with `--json`.
- An agent-ready rollout runbook (`rollout/`) and Claude Code skills (`skills/`).
