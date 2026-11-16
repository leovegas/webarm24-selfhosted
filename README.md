# WebArm24 Self-hosted

Run the complete [WebArm24](https://webarm24.online) automation platform on your own server: pipelines, workflows, browser agents, `ai_agent` steps with skills and MCP servers, the MCP endpoint for coding agents, and sandboxed Python steps. Your data, prompts and credentials never leave your infrastructure.

This repository is the **install kit**. It holds the Docker Compose stack, the `webarmctl` operations tool, the full documentation, an agent-ready rollout runbook and Claude Code skills. It contains **no application code**. The application ships as a licensed Docker image from `registry.webarm24.online`.

**[Pricing and purchase →](https://webarm24.online/self-hosted)** · **[Documentation →](docs/README.md)** · **[Changelog →](docs/CHANGELOG.md)**

---

## How licensing works

- **Buy once, run forever.** A one-time license runs every release built during its update window (30 days included) and keeps running after that, indefinitely.
- **Optional Updates subscription.** While it is active, every new release is yours. If you cancel, you keep the last version you were entitled to.
- **Priced by seats**, meaning user accounts on your install. Pipelines, runs and executions are unlimited.
- **Works offline.** The license key is checked on your server without contacting us. An optional weekly check-in renews keys automatically and sends only the license id, install id, version and user count ([what leaves your network](docs/00-overview.md)).

## Quickstart

You need a Linux x86_64 server with Docker Engine 24+ and Compose v2.20+ (2 vCPU, 4 GiB RAM and 40 GiB disk for an evaluation; 4 vCPU and 8 GiB for a team) ([requirements](docs/01-requirements.md)), plus a DNS name pointing at it and a license key.

```bash
sudo mkdir -p /opt/webarm && sudo chown "$USER" /opt/webarm
git clone --branch stable https://github.com/leovegas/webarm24-selfhosted.git /opt/webarm
cd /opt/webarm

./webarmctl preflight                       # checks Docker, kernel sandbox support, RAM, disk, ports, DNS
cp ~/license.key ./license.key              # from your purchase e-mail
PUBLIC_HOST=automation.example.com ADMIN_EMAILS=you@example.com LICENSE_FILE=./license.key ./webarmctl install
# → open https://automation.example.com/setup and create the admin account
```

The step-by-step guide, with the expected output of every command, is [docs/02-install.md](docs/02-install.md).

### Let an AI agent do the install

Open Claude Code, or any coding agent with a shell, in the cloned directory and say:

> Read AGENTS.md and rollout/ROLLOUT.md, then install WebArm24 on this server.

- [`rollout/ROLLOUT.md`](rollout/ROLLOUT.md) is a deterministic runbook. Every step has a precondition, a command, an expected result and a rollback. It stops for a human at DNS, secrets and destructive actions.
- [`rollout/rollout.yaml`](rollout/rollout.yaml) holds the same steps in machine-readable form.
- [`skills/`](skills/) contains Claude Code skills for install, upgrade, backup and restore, troubleshooting, license and configuration. They are also under `.claude/skills/`, so Claude Code finds them automatically.
- Every `webarmctl` command accepts `--json` and has stable exit codes.

## Day 2

```bash
./webarmctl status            # health, license, version, running work
./webarmctl doctor            # diagnose problems; every failing check prints a fix
./webarmctl backup            # database + stored files + .env + license → backups/*.tgz
./webarmctl upgrade 1.2.0     # backup, drain running work, switch, smoke test, automatic rollback on failure
./webarmctl support-bundle    # redacted diagnostics to send to support
```

Updating this kit (new `webarmctl`, compose file or docs) is `git pull`. Your `.env`, `license/`, `backups/` and `custom/` are ignored by git and never touched. See [docs/08-upgrades.md](docs/08-upgrades.md).

## What's in this repository

| Path | What it is |
|---|---|
| `webarmctl` | Operations CLI in bash: install, upgrade, backup, restore, doctor, license and more |
| `docker-compose.yml`, `Caddyfile.template`, `.env.example` | The stack: app, PostgreSQL 16 and Caddy (automatic HTTPS) |
| `docs/` | Documentation 00–13: requirements, configuration reference, TLS, AI providers, security, air-gapped installs, troubleshooting |
| `rollout/` | Agent-ready rollout runbook and checklists (go-live, upgrade, disaster-recovery drill) |
| `skills/`, `.claude/skills/` | Claude Code skills for operating the install |
| `AGENTS.md`, `CLAUDE.md` | Operating rules for AI agents working in this directory |
| `custom/` | Put your own `terms.html` / `privacy.html` here |

Branches and tags: `stable` is the latest release, and `vX.Y.Z` tags match image tags. Each [GitHub Release](../../releases) carries the same kit as a tarball, plus the release notes.

## Support

- **Questions, install problems, documentation bugs:** [open an issue](../../issues/new/choose). Never paste your license key, `.env` or unredacted logs. `./webarmctl support-bundle` redacts secrets for you.
- **Licensed customers:** e-mail support@webarm24.online with your license id (`lic_…`) and the support bundle.
- **Security vulnerabilities:** see [SECURITY.md](SECURITY.md). Please don't report them in public issues.
- **Evaluation key:** e-mail support@webarm24.online.

## License

The files in this repository (the install kit, `webarmctl`, documentation and skills) are licensed under the [MIT License](LICENSE).
The WebArm24 Self-hosted **application image** is commercial software. Its use is governed by the WebArm24 Self-hosted license agreement you accept at purchase, and it requires a valid license key.
