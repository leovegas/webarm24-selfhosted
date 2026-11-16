# 07 — E-mail

SMTP is optional but recommended. It is used for:
- password resets and e-mail verification;
- workflow **approval** e-mails, which contain links to approve or reject;
- `email` steps in pipelines, and the agent's `send_email` tool.

```bash
SMTP_HOST=smtp.example.com
SMTP_PORT=587               # STARTTLS
SMTP_USER=automation@example.com
SMTP_PASSWORD=…
SMTP_FROM=Automation <automation@example.com>
```
Run `./webarmctl restart`.

## Without SMTP

- New accounts are marked verified automatically, because the admin controls sign-ups.
- Password reset links are written to the app log instead of e-mailed. Get them with:
  ```bash
  ./webarmctl logs app | grep -i "reset"
  ```
- Approval and `email` steps fail with an SMTP error.

## Test

1. Sign out.
2. Click **Forgot password** with your admin address.
3. The e-mail should arrive within a minute. If it doesn't, run `./webarmctl logs app | grep -i smtp`.

## Limiting where pipelines can send

```bash
AGENT_EMAIL_ALLOW_DOMAINS=example.com,partner.com
AGENT_EMAIL_ALLOW_ADDRESSES=alerts@vendor.io
AGENT_EMAIL_MAX_RECIPIENTS=20
```

## Providers

- Microsoft 365: `smtp.office365.com:587`. Authenticated SMTP must be enabled for the mailbox.
- Google Workspace: `smtp.gmail.com:587` with an app password, or use the SMTP relay.
- Amazon SES, Postmark, Mailgun and SendGrid: use their SMTP credentials on port 587.
