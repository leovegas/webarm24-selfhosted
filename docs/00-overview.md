# 00 — Overview

## What you get

| Part | What it is |
|---|---|
| **Image** `registry.webarm24.online/webarm/selfhosted:<version>` | The whole application: web UI, REST + WebSocket API, MCP endpoint, scheduler, workflow engine, headless Chromium, the bubblewrap Python sandbox, ffmpeg, fonts and the map-video library. The backend is compiled to native code. |
| **Bundle** `webarm-selfhost-bundle-<version>.tgz` | `docker-compose.yml`, `Caddyfile.template`, `.env.example`, `webarmctl`, these docs, the rollout runbook and agent skills. Plain text you can read and audit. |
| **License key** `license.key` (`WA1.…`) | A signed file stating your seats and your update window. It is verified offline. |
| **Registry login** | Username = license id (`lic_…`), password = registry token from your purchase e-mail. You can rotate it on your account page. |

## Architecture

```
                 users / browsers / coding agents (MCP)
                                │ HTTPS 443
                     ┌──────────▼──────────┐
                     │ caddy               │  automatic TLS (Let's Encrypt or internal CA)
                     └──────────┬──────────┘
                                │ http :8000 (internal network only)
 ┌──────────────────────────────▼───────────────────────────────────┐
 │ app (WebArm24 image)                                             │
 │  • UI (/pipelines, /workflows, /admin, /setup)  • REST /api/*    │
 │  • WebSocket chat /ws  • MCP /mcp (API keys)  • inbound webhooks │
 │  • scheduler (cron, run_at) • workflow engine • agent loop       │
 │  • Playwright Chromium • bubblewrap sandbox for Python steps     │
 │  • egress proxy for sandboxed steps (public internet only)       │
 └───────┬───────────────────────────────┬──────────────────────────┘
         │ postgres :5432                │ volume appdata (stored files, python envs)
 ┌───────▼────────┐                      ▼
 │ db (Postgres16)│  volume pgdata
 └────────────────┘
```

Volumes: `pgdata` holds all records. `appdata` holds stored files, saved state and package caches. `knowledge` is a per-site crawl cache that rebuilds itself. `videos` holds optional recordings. `caddy_data` holds the certificates. Back up `pgdata` and `appdata`, plus `.env` and `license/`. `webarmctl backup` does all of that.

## What leaves your network

| Traffic | When | Contains | Turn off |
|---|---|---|---|
| License check-in → `https://webarm24.online/api/selfhost/checkin` | weekly (from the scheduler leader) | license id, random install id, version, build date, number of user accounts | `LICENSE_PHONE_HOME=false` |
| Image pulls → `registry.webarm24.online` | only when you run `install` / `upgrade` | your registry credentials | load images from a tarball instead ([13](13-air-gapped.md)) |
| AI provider APIs | when pipelines/agents run | prompts and data of the runs that use AI | use a gateway or private endpoint ([06](06-ai-providers.md)) |
| Let's Encrypt | certificate issue/renewal | your domain name | `TLS_MODE=internal` or your own certificates ([05](05-domain-tls.md)) |
| Whatever your pipelines fetch | by design | — | egress settings ([10](10-security.md)) |
| Browser UI assets: Google Fonts and the code editor from cdnjs | when users open the UI | standard browser requests | see [13](13-air-gapped.md) |

Nothing else is sent. There is no telemetry about users, pipelines, runs or data, and no analytics in the UI. The self-hosted build removes them at build time.

## The license model in one paragraph

You **buy once** and keep running every release built during your update window forever. The window is the first 30 days after purchase. The optional **Updates** subscription moves the window forward every month while it is active, with a 7-day grace period. If you stop it, nothing stops working: you keep every version built before your window ended. A newer image refuses to start until you renew (`WA-LIC-002`), and you go back to your last eligible tag with `webarmctl rollback`. **Seats** cap the number of user accounts. Existing users are never locked out. Details: [04 — License](04-license.md).

## Differences from webarm24.online

- **No credits.** Every action is free and unmetered. You can enable credits with `CREDITS_ENABLED=true` if you want internal quotas.
- **Sign-in is required** for the API and UI. Seats count accounts. Inbound webhooks and e-mailed approval links keep working with their own tokens.
- **No limit on automations per user.** The SaaS alpha quota does not apply.
- **Without SMTP, new accounts are verified automatically**, because the admin controls who signs up.
- **Branding:** `BRAND_NAME`, `BRAND_LOGO_URL`, `SUPPORT_EMAIL`. The terms and privacy pages are yours: put `terms-of-service.html` / `privacy-policy.html` in `custom/`.
- Removed: marketing pages, SEO metadata, analytics, the cookie banner and the credit shop.
