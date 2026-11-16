# WebArm24 Self-hosted — documentation

Run the complete WebArm24 automation platform on your own server: pipelines, workflows, browser agents, `ai_agent` steps, the MCP server for coding agents, and the sandboxed Python steps. It ships as a private Docker image and a small bundle of operations files. No source code is involved, and you can run it without internet access.

| If you want to… | Read |
|---|---|
| Understand what you are installing and what leaves your network | [00 — Overview](00-overview.md) |
| Size and prepare a server | [01 — Requirements](01-requirements.md) |
| Install from zero in ~20 minutes | [02 — Install](02-install.md) |
| Look up a setting | [03 — Configuration reference](03-configuration.md) |
| Activate, renew or troubleshoot the license | [04 — License](04-license.md) |
| Use your own domain, certificate or reverse proxy | [05 — Domain & TLS](05-domain-tls.md) |
| Connect Anthropic / OpenAI / OpenRouter | [06 — AI providers](06-ai-providers.md) |
| Send e-mail | [07 — E-mail](07-email.md) |
| Upgrade or roll back | [08 — Upgrades](08-upgrades.md) |
| Back up and restore | [09 — Backup & restore](09-backup-restore.md) |
| Harden the install | [10 — Security](10-security.md) |
| Monitor and tune | [11 — Operations](11-operations.md) |
| Fix a problem | [12 — Troubleshooting](12-troubleshooting.md) |
| Run without internet | [13 — Air-gapped install](13-air-gapped.md) |
| See what changed | [CHANGELOG](CHANGELOG.md) |

**Letting an AI agent do it.** The bundle is written so that Claude Code, or any coding agent with a shell on the server, can do the rollout:
- `rollout/ROLLOUT.md` is a deterministic runbook. Every step has a precondition, a command, an expected result and a rollback.
- `rollout/rollout.yaml` holds the same steps in machine-readable form.
- `skills/` contains Claude Code skills for install, upgrade, backup, troubleshooting, license and configuration.
- `AGENTS.md` / `CLAUDE.md` give operating rules for agents.
- Every `webarmctl` command accepts `--json`.

Start with: *"Read AGENTS.md and rollout/ROLLOUT.md, then install WebArm24 on this server."*
