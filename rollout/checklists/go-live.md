# Go-live checklist

- [ ] `./webarmctl doctor --json` → `"ok": true`
- [ ] `./webarmctl smoke` → all checks passed (including the sandboxed Python step)
- [ ] `https://PUBLIC_HOST` has a valid certificate (or the internal CA is distributed to clients)
- [ ] License **valid**; seats and `updates_until` noted (`/setup`)
- [ ] At least one AI provider key; a test `ai_agent` step answered
- [ ] SMTP tested (Forgot password e-mail arrives), or the "no SMTP" behaviour has been accepted
- [ ] `ADMIN_EMAILS` holds at least two people
- [ ] Backup cron installed; first backup copied off-server **encrypted**; `.env` stored in the password vault
- [ ] Firewall: only 80/443 (+22 from admin IPs) inbound
- [ ] `PIPELINE_PYTHON_NETWORK=proxy` (or `none`); egress allow list decided
- [ ] Monitoring on `https://PUBLIC_HOST/ready` (keyword `"license":"valid"`)
- [ ] Legal pages in `custom/` if users outside your company sign in
- [ ] Upgrade policy written down: pinned tag, who upgrades, when (docs/08)
- [ ] Users invited; MCP endpoint `https://PUBLIC_HOST/mcp` shared with developers
