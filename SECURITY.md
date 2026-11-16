# Security policy

## Reporting a vulnerability

E-mail **security@webarm24.online**. Please include:
- the affected version (`./webarmctl version`);
- what an attacker can do, and the steps to reproduce it;
- whether it is already public.

Please **don't open a public issue** for security problems. We acknowledge reports within 3 working days and tell you when a fix will ship. Fixes for supported versions are published as patch releases and called out in the **Security** section of the [changelog](docs/CHANGELOG.md).

## Supported versions

Security fixes go into the latest minor release. Customers whose update window has ended can still get a security patch built for their window. Contact support.

## Hardening

See [docs/10-security.md](docs/10-security.md). It covers the Python step sandbox, egress modes, secrets rotation and a hardening checklist.

## Never share in issues

License keys (`WA1.…`), `.env`, database dumps, backups, registry passwords and unredacted logs. `./webarmctl support-bundle` produces a redacted bundle that is safe to e-mail to support.
