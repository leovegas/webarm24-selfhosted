# 10 — Security

## Threat model in short

Users run code. Python and JS steps, agents that browse the web, and HTTP steps are all part of the product. The install keeps that code away from the host, the database and your network, and keeps tenants apart through accounts.

| Layer | Protection |
|---|---|
| Python steps | **bubblewrap** sandbox per execution: new user/PID/IPC/network namespaces, read-only `/usr`, private `/work`, CPU/memory/file-size/time limits, no access to the app's files or environment |
| Python network | `PIPELINE_PYTHON_NETWORK=proxy`: a per-execution Unix-socket proxy that resolves names itself and refuses private, loopback, CGNAT, link-local/metadata (169.254.169.254), ULA and NAT64 targets. It also enforces port, host, rate, byte and connection limits, and logs each connection with the run id. `none` = no network at all. **Avoid `host`**: it can reach your DB and LAN |
| JS steps | embedded V8 (mini-racer): no filesystem, no network |
| `ai_agent` web tools | private/loopback IPs are blocked on every redirect hop unless `AI_AGENT_ALLOW_PRIVATE_NETWORK=true` |
| Accounts | bcrypt passwords and JWT sessions; seats = accounts; sign-in required for API/UI (`SELFHOST_ALLOW_ANONYMOUS=false`) |
| Secrets at rest | pipeline variables, trigger tokens and saved browser logins are Fernet-encrypted with `PIPELINE_FIELD_ENCRYPTION_KEY` |
| MCP | per-user API keys (`wak_…`), only sha256 hashes are stored; keyless requests can only do discovery |
| Transport | Caddy TLS 1.2+/1.3, HSTS |

The app container needs `SYS_ADMIN`/`NET_ADMIN` and unconfined seccomp/AppArmor so that bubblewrap can create namespaces **inside** it. The container itself is not privileged, has no host mounts beyond its volumes, and has no Docker socket.

## Hardening checklist

- [ ] Firewall: only 80/443 inbound (22 for SSH, from admin IPs). Postgres is not published.
- [ ] `.env` is mode 600, owned by the operator account; `license/` and `backups/` are mode 700.
- [ ] `PIPELINE_PYTHON_NETWORK=proxy` (bundle default) or `none`; set `PIPELINE_PYTHON_EGRESS_ALLOW_HOSTS` if your use cases have a known list of APIs.
- [ ] `AI_AGENT_ALLOW_PRIVATE_NETWORK=false` unless agents must reach the intranet, and then only on a dedicated install.
- [ ] `AGENT_EMAIL_ALLOW_DOMAINS` set if pipelines must not e-mail arbitrary addresses.
- [ ] SSO: enable Google sign-in for your Workspace domain, or keep password accounts with strong passwords. Remove leavers at `/admin`.
- [ ] Encrypted off-site backups and a tested restore ([09](09-backup-restore.md)).
- [ ] Upgrades on a schedule ([08](08-upgrades.md)): security fixes ship in patch releases.
- [ ] Logs shipped to your SIEM (`docker logs` JSON driver → your collector); egress proxy lines contain the run id.

## Rotating secrets

| Secret | How | Effect |
|---|---|---|
| `JWT_SECRET` | new value in `.env`, `./webarmctl restart` | everyone signs in again; MCP keys keep working |
| `POSTGRES_PASSWORD` | `docker compose -p webarm exec db psql -U webarm -c "ALTER USER webarm PASSWORD '<new>'"`, update `.env`, restart | none |
| `PIPELINE_FIELD_ENCRYPTION_KEY` | **do not rotate in place** — encrypted data would become unreadable. Contact support for the re-encryption procedure | — |
| Registry password | **Your licenses** → *New registry password*, then `docker login` again | old token stops immediately |
| User MCP keys | each user revokes and creates keys in the **MCP** dialog | immediate |

## Reporting a vulnerability

E-mail security@webarm24.online with the details. Please don't open public issues.
