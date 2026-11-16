# 04 — License

## How it works

Your key is a signed text file: `WA1.<payload>.<signature>`. The app verifies the signature offline with the vendor public key compiled into the image. The payload holds:

| Field | Meaning |
|---|---|
| `lic_id` | your license id (also your registry username) |
| `customer`, `email` | who it is licensed to |
| `seats` | maximum number of user accounts |
| `updates_until` | last day of your update window |
| `issued_at` | when this key was issued (newer keys replace older ones) |

**The rule:** an image runs if its build date is on or before `updates_until`.
- Buying a license gives you 30 days of updates. Every image built in that time is yours forever.
- While the monthly **Updates** subscription is active, each paid month moves `updates_until` to the end of the period + 7 days, and a new key is issued.
- If you cancel, nothing breaks. You keep running your current version and can reinstall any tag built before the window ended.
- The image's build date is printed by `./webarmctl version` and stored in the image label `webarm.build_date`.

## Activate

Use one of these. They are equivalent; if several keys are present, the one with the newest `issued_at` wins.

1. **Setup page:** `https://PUBLIC_HOST/setup` → sign in as an admin → paste the key → **Activate**.
2. **Key file:** `./webarmctl license set ./license.key`. This writes `license/webarm_license` and restarts the app.
3. **Environment:** `LICENSE_KEY=WA1.…` in `.env` (for secret managers and CI). Then run `./webarmctl restart`.

Check the result:
```bash
./webarmctl license status        # OK license — valid
./webarmctl license status --json # {"ok":true,"license":"valid",...}
```
The admin view (`/setup`, or `/admin` → **License**) shows seats used, the update window, key source, install id and the last check-in.

## Renewals

- **With phone-home on** (the default): the app checks in once a week and installs the renewed key automatically. To do it right now: `/setup` → **Check for a renewed key now**.
- **Air-gapped:** download `license.key` from **Your licenses** on https://webarm24.online/self-hosted after each renewal, then run `./webarmctl license set license.key`.

## Seats

The number of user accounts may not exceed `seats`.
- At the limit, new sign-ups (password or Google) get **403 `WA-LIC-005`**.
- Existing users keep working. Nothing is ever locked.
- To free a seat, delete a user at `/admin`.
- To get more seats, buy a bigger plan, or contact sales for an upgrade key.

## Error codes

| Code | State | What it means | What to do |
|---|---|---|---|
| <a id="WA-LIC-001"></a>`WA-LIC-001` | missing | No key configured. The app is in **setup mode**: only `/setup`, sign-in and health answer | Paste the key at `/setup` or run `./webarmctl license set license.key` |
| <a id="WA-LIC-002"></a>`WA-LIC-002` | outdated | This image was built after `updates_until` | Renew Updates and refresh the key (`/setup` → check in, or download it again), **or** go back to an eligible tag: `./webarmctl rollback` / `./webarmctl upgrade <older tag>` |
| <a id="WA-LIC-003"></a>`WA-LIC-003` | invalid | Malformed key or bad signature (truncated copy, wrong file, key for another product) | Download `license.key` again; don't edit it. Whitespace and line breaks are fine |
| <a id="WA-LIC-004"></a>`WA-LIC-004` | revoked | The vendor revoked the license for builds after a date | Contact the vendor |
| <a id="WA-LIC-005"></a>`WA-LIC-005` | (valid) | Seat limit reached; only new accounts are refused | Remove a user or buy more seats |

In setup mode, API calls return **503** with `{"detail":{"code":"WA-LIC-00x","message":"…"}}`, and browser page requests redirect to `/setup`. `/ready` stays **200** so your load balancer keeps routing admins to the setup page. The `license` field in `/ready` says why.

## Moving to a new server

The license is not bound to a machine. Restore a backup on the new server ([09](09-backup-restore.md)) and shut down the old one. Running more installs than you bought violates the license terms. The check-in reports the install id, so the vendor can see duplicates.
