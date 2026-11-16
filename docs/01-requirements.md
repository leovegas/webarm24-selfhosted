# 01 — Requirements

## Server

| Users / load | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Evaluation, ≤5 users, a few scheduled pipelines | 2 | 4 GiB | 40 GiB SSD | one browser session at a time is comfortable |
| Team, ≤25 users, tens of daily runs | 4 | 8 GiB | 80 GiB SSD | default settings |
| Department, ≤100 users, parallel workflows, video steps | 8 | 16–32 GiB | 200 GiB SSD | raise `WORKFLOW_MAX_CONCURRENCY`, `PIPELINE_PYTHON_MEMORY_MAX_BYTES` |

Rules of thumb:
- Each concurrent browser session needs about 300–600 MiB.
- Each Python step can use up to `PIPELINE_PYTHON_MEMORY_MAX_BYTES` (default 512 MiB). Video encodes need 1–4 GiB.
- The image is about 6 GiB on disk.
- Plan disk space for stored files plus about 3 days of backups.

## Operating system

- **Linux x86_64 (amd64)**. Tested: Ubuntu 22.04 / 24.04 LTS, Debian 12, Rocky/Alma 9. ARM is not supported yet.
- **Docker Engine 24+** with the **Compose v2 plugin** (`docker compose version`). Rootless Docker is not supported, because the Python sandbox needs `SYS_ADMIN` inside the container.
- **Kernel user namespaces enabled.** This is the default on all of the above. Check with:
  ```bash
  cat /proc/sys/user/max_user_namespaces           # must be > 0
  cat /proc/sys/kernel/unprivileged_userns_clone   # 1 or file absent
  ```
- **Ubuntu 24.04:** AppArmor restricts unprivileged user namespaces. The compose file runs the app container with `apparmor=unconfined`, which is enough. If you remove that line, Python steps fail (see [12](12-troubleshooting.md#sandbox)).
- SELinux (RHEL family) in enforcing mode works with Docker's default policy. If bubblewrap fails, see [12](12-troubleshooting.md#sandbox).

## Network

| Direction | Port | Why |
|---|---|---|
| inbound | 80/tcp | ACME HTTP challenge + redirect to HTTPS (not needed with `TLS_MODE=internal` or your own proxy) |
| inbound | 443/tcp+udp | the app (HTTP/2 + HTTP/3) |
| outbound | 443 | AI providers, registry pulls, license check-in, whatever your pipelines access |
| outbound | 25/465/587 | your SMTP server (optional) |

DNS: an A/AAAA record for `PUBLIC_HOST` pointing at the server. You need it for automatic certificates.

## Accounts and keys you need

- The license key file and the registry credentials (purchase e-mail, or **Your licenses** on https://webarm24.online/self-hosted).
- At least one AI provider key: Anthropic, OpenAI, or OpenRouter.
- Optional: SMTP credentials, and a Google OAuth client for "Sign in with Google".

`./webarmctl preflight` checks everything on this page that can be checked automatically.
