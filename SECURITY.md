# Security Policy

## Reporting a Vulnerability

Please report security issues to **<ahmedmounib2@gmail.com>** rather than opening a public GitHub
issue.

---

## Credential Revocation Log

### 2026-05-20 — Localhost TLS key revoked (C-1)

A mkcert-generated private key (`cert/localhost+2-key.pem`) and its corresponding certificate
(`cert/localhost+2.pem`) were accidentally committed and have been removed from git tracking.

**Impact:** Local-only development certificates. The private key was used exclusively for
`localhost`, `127.0.0.1`, and `::1`. It was not used in any production environment and carries no
external trust (signed by the local mkcert CA, not a public CA).

**Action taken:**

- Files removed from git index via `git rm --cached`.
- `cert/*.pem` and `cert/*.key` added to `.gitignore`.
- Developers must regenerate certs locally; see `cert/README.md`.
- Git history was **not** rewritten to avoid disrupting existing checkouts. Because the key was
  local-only and not signed by a trusted CA, history rewrite was assessed as unnecessary.

---

### 2026-05-20 — SESSION_SECRET rotation recommended (C-2)

A `cookies.txt` file (Netscape cookie jar format) was accidentally committed and has been removed
from git tracking.

**Impact:** The file may have contained session or CSRF cookies issued against the development
environment. Even if expired, the presence of the file reveals domain, path, and auth-scheme
details.

**Action required (next production deployment):**

- Rotate `SESSION_SECRET` in all environments (development, staging, production).
- Rotating the secret invalidates all existing sessions and CSRF tokens; coordinate with active
  users to avoid unexpected logouts.
- No immediate action is required if the exposed cookies were issued against `localhost` only.

---

## Dependency Updates

Keep dependencies up to date. Run `npm audit` regularly and address high/critical advisories before
each production release.
