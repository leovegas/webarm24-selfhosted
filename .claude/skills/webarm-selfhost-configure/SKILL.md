---
name: webarm-selfhost-configure
description: Change the configuration of a WebArm24 Self-hosted install in .env. Covers AI providers and models, SMTP, Google sign-in, branding, TLS mode and your own reverse proxy, Python step network/egress/limits/packages, storage limits and workflow concurrency. Use when the user wants to connect an AI key, set up e-mail, rebrand, change the domain, tune limits, allow Python steps to reach an API, or put WebArm24 behind their proxy.
---

# Configure WebArm24 Self-hosted

Every setting is a line in `.env`. The full reference is `docs/03-configuration.md`. Apply changes with `./webarmctl restart --json`, then `./webarmctl doctor --json`.

## How to edit `.env` safely
- Back it up first: `cp .env .env.bak.$(date +%s)` (mode 600).
- Set a value without echoing secrets: keep the value in an env var and use `sed -i "s|^KEY=.*|KEY=${VALUE}|" .env`. If the key isn't present yet, append it with `printf 'KEY=%s\n' "$VALUE" >> .env`.
- **Never change** `PIPELINE_FIELD_ENCRYPTION_KEY`: encrypted data would become unreadable. Never change `POSTGRES_PASSWORD` by editing `.env` alone (see docs/10, rotating secrets).
- Don't set the compose-derived variables (`DATABASE_URL`, `PUBLIC_BASE_URL`, `LICENSE_KEY_FILE`, …).

## Recipes
| Goal | Lines | Notes |
|---|---|---|
| Anthropic | `AI_PROVIDER=anthropic`, `ANTHROPIC_API_KEY=…` | docs/06 |
| OpenAI default | `AI_PROVIDER=openai`, `OPENAI_API_KEY=…` | |
| More `ai_agent` models | `OPENROUTER_API_KEY=…`; optional `AI_AGENT_MODELS=[…]` | |
| Internal LLM gateway | `OPENROUTER_BASE_URL=https://gw/v1`, `OPENROUTER_API_KEY=…`, `AI_AGENT_MODELS=[{"provider":"openrouter","id":"…","label":"…","tier":"standard"}]` | |
| E-mail | `SMTP_HOST/PORT/USER/PASSWORD/FROM` | test: Forgot password |
| Google sign-in | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`; redirect `https://PUBLIC_HOST/api/auth/google/callback` | |
| Branding | `BRAND_NAME`, `BRAND_LOGO_URL`, `SUPPORT_EMAIL`; legal pages in `custom/` | |
| LAN certificate | `TLS_MODE=internal` | distribute the root CA (docs/05) |
| Own proxy | `HTTP_PORT`/`HTTPS_PORT`, or a compose override exposing the app | docs/05. **Ask** before changing the network exposure |
| Python steps call one API | `PIPELINE_PYTHON_NETWORK=proxy`, `PIPELINE_PYTHON_EGRESS_ALLOW_HOSTS=api.example.com` | security decision → **ask** |
| Long / heavy Python | `PIPELINE_PYTHON_STEP_TIMEOUT_SEC=600`, `PIPELINE_PYTHON_MEMORY_MAX_BYTES=4294967296`, `PIPELINE_PYTHON_FILE_MAX_BYTES=…` | needs RAM |
| Private PyPI | `PIPELINE_PYTHON_PACKAGES_INDEX_URL=https://pypi.internal/simple` | |
| Agents reach intranet | `AI_AGENT_ALLOW_PRIVATE_NETWORK=true` | security decision → **ask** |
| Bigger files | `USER_FILES_TOTAL_MAX_BYTES`, `USER_FILES_FILE_MAX_BYTES` | disk |
| More parallel workflow nodes | `WORKFLOW_MAX_CONCURRENCY=8` | RAM/CPU |
| Restrict e-mail recipients | `AGENT_EMAIL_ALLOW_DOMAINS=…` | |
| Change domain | `PUBLIC_HOST=…` | then DNS, OAuth redirect, webhook URLs (docs/05) |

## Ask the human before
- anything security-relevant: `PIPELINE_PYTHON_NETWORK=host`, private-network access, `SELFHOST_ALLOW_ANONYMOUS`, exposing ports;
- the domain or TLS;
- turning on `CREDITS_ENABLED`.

## Verify
`./webarmctl restart --json` → `./webarmctl doctor --json` → report the changed lines. Name the variables, never the secret values.
