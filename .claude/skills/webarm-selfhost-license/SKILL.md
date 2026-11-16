---
name: webarm-selfhost-license
description: Explain and manage the WebArm24 Self-hosted license, covering activation, seat limits, the update window (updates_until vs. image build date), renewals through the monthly Updates subscription, air-gapped key refresh, phone-home and WA-LIC error codes. Use when the user asks about the license key, seats, "can we upgrade", renewal, expiry, WA-LIC-00x errors, or why the app is in setup mode.
---

# WebArm24 Self-hosted license

## Model (explain it this way)
- **One-time purchase.** You run every image built on or before `updates_until` forever. A purchase includes 30 days of updates.
- **Monthly Updates subscription (optional).** Each paid month moves `updates_until` to the end of the period + 7 days, and the vendor issues a new key.
- **Cancelling** never stops the install. Newer images refuse to start (`WA-LIC-002`), so you stay on an eligible tag.
- **Seats** = the maximum number of user accounts. At the limit only *new* sign-ups are refused (`WA-LIC-005`).
- The key (`WA1.…`) is verified **offline** against a public key compiled into the image. Editing the key breaks its signature.

## Check
```bash
./webarmctl license status --json     # .license: valid|missing|invalid|outdated|revoked, .code
./webarmctl version --json            # running version; the build date is the image label webarm.build_date
```
Full details (seats used, `updates_until`, source, install id, last check-in) are on `https://PUBLIC_HOST/setup` for admins, or at `GET /api/license/status` with an admin JWT. They are also available through the MCP tool `license_status`.

## Activate or replace a key
- File: `./webarmctl license set /path/license.key --json` (writes `license/webarm_license`, restarts, verifies).
- UI: `/setup` → paste → **Activate**.
- Env: `LICENSE_KEY=…` in `.env`, then restart.
- When several sources are present, the key with the newest `issued_at` wins.

## Renewal
- Phone-home on (`LICENSE_PHONE_HOME=true`, the default): a weekly check-in installs the renewed key automatically. To force it: `/setup` → **Check for a renewed key now**.
- The check-in sends only: license id, install id, version, build date, number of accounts.
- Air-gapped: the human downloads the new `license.key` from webarm24.online/self-hosted → `./webarmctl license set`.

## Can we upgrade to version X?
Compare X's build date (docs/CHANGELOG.md, or `docker image inspect -f '{{ index .Config.Labels "webarm.build_date" }}' <image>`) with `updates_until`. If build date ≤ `updates_until`, yes. Otherwise the customer has to renew first. `./webarmctl upgrade` performs this check before changing anything.

## Error codes
`WA-LIC-001` missing · `WA-LIC-002` image newer than the window · `WA-LIC-003` bad key/signature · `WA-LIC-004` revoked for this version · `WA-LIC-005` seats full. Details and fixes: `docs/04-license.md#error-codes`.

## Never
- Never try to modify, forge or bypass the license, the edition or the image. Doing so violates the license agreement, and the key is cryptographically signed anyway.
- Never paste the key into chats, tickets or commits. Treat it like a password.
