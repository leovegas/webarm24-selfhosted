# 03 — Configuration reference

All configuration lives in `.env` next to `docker-compose.yml`. After any change run `./webarmctl restart`.
Values marked **compose** are set by `docker-compose.yml` from other values. Don't set those in `.env`.

## Core

| Variable | Default | Purpose |
|---|---|---|
| `PUBLIC_HOST` | — (**required**) | DNS name. It drives certificates, links in e-mails and OAuth redirects |
| `ADMIN_EMAILS` | — (**required**) | Comma-separated. These accounts are admins, and before activation only they can sign up |
| `POSTGRES_PASSWORD` | generated | Database password. Letters and digits only |
| `JWT_SECRET` | generated | Signs sessions. Changing it signs everyone out |
| `PIPELINE_FIELD_ENCRYPTION_KEY` | generated | Fernet key for stored secrets. **Never change or lose it** |
| `WEBARM_ENV` | `production` | Leave as is |
| `WEBARM_IMAGE` / `WEBARM_TAG` | registry / `stable` | Image to run. Change the tag only via `webarmctl upgrade <tag>` |
| `DATABASE_URL`, `PUBLIC_BASE_URL`, `LICENSE_KEY_FILE`, `PIPELINE_USER_FILES_ROOT`, `PIPELINE_PYTHON_WORKDIR_ROOT`, `CUSTOM_PAGES_DIR` | **compose** | Derived. Don't override |

## License

| Variable | Default | Purpose |
|---|---|---|
| `LICENSE_PHONE_HOME` | `true` | Weekly check-in that picks up renewed keys. `false` for air-gapped installs |
| `LICENSE_SERVER_URL` | `https://webarm24.online` | Only change it if the vendor tells you to |
| `LICENSE_KEY` | — | Alternative to the key file (the highest `issued_at` among all sources wins) |

## AI

| Variable | Default | Purpose |
|---|---|---|
| `AI_PROVIDER` | `anthropic` | Provider of the chat / pipeline `agent` steps: `anthropic` or `openai` |
| `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `OPENROUTER_API_KEY` | — | Provider keys. Every `ai_agent` model whose key is set becomes selectable |
| `CLAUDE_MODEL` / `OPENAI_MODEL` | built-in | Model for `agent` steps and chat |
| `AI_AGENT_MODELS` | built-in catalog | JSON array that adds or overrides `ai_agent` models (`[{"provider":"openrouter","id":"…","label":"…","tier":"standard"}]`) |
| `AGENT_MAX_OUTPUT_TOKENS` | `8192` | Completion budget per LLM turn |
| `ANTHROPIC_PROMPT_CACHE` / `ANTHROPIC_PROMPT_CACHE_TTL` | `true` / `5m` | Prompt caching (it lowers cost) |
| `LLM_REQUESTS_PER_MINUTE` | `480` | Global rate limit towards providers |
| `AI_AGENT_ALLOW_PRIVATE_NETWORK` | `false` | Let `ai_agent` web tools reach private IPs (your intranet) |

## E-mail and sign-in

| Variable | Default | Purpose |
|---|---|---|
| `SMTP_HOST` `SMTP_PORT` `SMTP_USER` `SMTP_PASSWORD` `SMTP_FROM` | — / 587 | Outgoing mail (STARTTLS) — [07](07-email.md) |
| `AGENT_EMAIL_ALLOW_DOMAINS` / `AGENT_EMAIL_ALLOW_ADDRESSES` / `AGENT_EMAIL_MAX_RECIPIENTS` | — / — / 20 | Restrict where pipelines may send mail |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | — | "Sign in with Google". Redirect URI `https://PUBLIC_HOST/api/auth/google/callback` |
| `SELFHOST_ALLOW_ANONYMOUS` | `false` | `true` lets visitors use the API without an account (not recommended) |
| `SUPPORT_NOTIFY_EMAIL` | empty | Address told about every new sign-up (empty = off) |

## Branding

| Variable | Default | Purpose |
|---|---|---|
| `BRAND_NAME` | `WebArm24` | Replaces the product name in the UI, the page titles and the MCP server name |
| `BRAND_LOGO_URL` | — | Logo image URL for the top bar |
| `SUPPORT_EMAIL` | empty | Shown to users in limit messages ("Contact …") |
| files in `custom/` | — | `terms-of-service.html`, `privacy-policy.html`, served at `/terms-of-service` and `/privacy-policy` |

## HTTPS / proxy

| Variable | Default | Purpose |
|---|---|---|
| `TLS_MODE` | empty | empty = Let's Encrypt; `internal` = Caddy's own CA (LAN) |
| `ACME_EMAIL` | — | Let's Encrypt contact |
| `HTTP_PORT` / `HTTPS_PORT` | 80 / 443 | Host ports Caddy binds |

## Python steps and sandbox

| Variable | Default | Purpose |
|---|---|---|
| `PIPELINE_PYTHON_NETWORK` | `proxy` (bundle) | `none` = no network. `proxy` = public internet through the filtering egress proxy (private, loopback and metadata IPs are blocked). `host` = unrestricted (it can reach your DB and LAN) |
| `PIPELINE_PYTHON_EGRESS_PORTS` | `80,443` | Allowed destination ports through the proxy |
| `PIPELINE_PYTHON_EGRESS_ALLOW_HOSTS` / `_DENY_HOSTS` | — | Optional host allow and deny lists |
| `PIPELINE_PYTHON_EGRESS_MAX_CONNECTIONS` / `_MAX_CONCURRENT` / `_RATE_PER_SEC` / `_MAX_BYTES` | 200 / 8 / 5 / 1 GiB | Limits per step execution |
| `PIPELINE_PYTHON_STEP_TIMEOUT_SEC` | `10` | Wall-clock limit per Python step (CPU limit ≈ timeout × cores). Raise it for media work |
| `PIPELINE_PYTHON_MEMORY_MAX_BYTES` | 512 MiB | Address-space limit incl. child processes (ffmpeg) |
| `PIPELINE_PYTHON_FILE_MAX_BYTES` | 10 MiB | Largest file a step may write |
| `PIPELINE_PYTHON_PACKAGES_ENABLED` | `true` | Per-step approved packages (`py_requirements`) |
| `PIPELINE_PYTHON_PACKAGES_INDEX_URL` / `_FIND_LINKS` / `_NO_INDEX` | PyPI | Private mirror or offline wheelhouse ([13](13-air-gapped.md)) |
| `PIPELINE_PYTHON_PACKAGES_ANY_VERSION` | `false` | Allow any exact version of an approved package |

## Storage limits

| Variable | Default | Purpose |
|---|---|---|
| `USER_FILES_TOTAL_MAX_BYTES` / `USER_FILES_FILE_MAX_BYTES` | 1 GiB / 200 MiB | Stored files per user and per file |
| `DATA_STORAGE_VALUE_MAX_BYTES` / `_MAX_STORES_PER_USER` / `_TOTAL_MAX_BYTES` | 256 KiB / 50 / 5 MiB | JSON/text stores |
| `SKILLS_MAX_PER_USER` | 100 | Agent skills per user |
| `PIPELINE_HISTORY_MERGE_SEC` / `PIPELINE_HISTORY_MAX_REVISIONS` | 300 / 200 | Version history |

## Scheduler, workflows, browser

| Variable | Default | Purpose |
|---|---|---|
| `SCHEDULER_ENABLED` | `true` | Cron / `run_at` triggers |
| `WORKFLOW_MAX_CONCURRENCY` | `4` | Parallel nodes per workflow run |
| `WORKFLOW_INLINE_WAIT_MAX_SEC` | `60` | Longer waits park the run and resume later |
| `WORKFLOW_FOR_EACH_MAX_ITEMS` | `200` | Max items per `for_each` |
| `PLAYWRIGHT_HEADLESS` | `true` | Leave on servers |
| `PLAYWRIGHT_VIDEO` | `false` | Record sessions (they show up under `/videos`) |
| `SCREENSHOT_WIDTH` | `900` | Screenshot width sent to the model |
| `AUTO_DISMISS_COOKIE_BANNER` | `true` | Click "Accept" on common consent banners |

## Metering (optional)

| Variable | Default | Purpose |
|---|---|---|
| `CREDITS_ENABLED` | `false` | Turn on usage metering with credits. There is no credit shop in the self-hosted image, so leave it off unless you have a plan for granting credits |
| `CREDIT_COST_*` | 1 | Cost per action when credits are enabled |
