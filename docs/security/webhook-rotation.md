# Webhook Secret Rotation

## Stripe Webhook Secret Rotation

Stripe supports up to 5 simultaneously active webhook secrets per endpoint, which allows
zero-downtime rotation.

1. In the Stripe Dashboard, go to **Developers → Webhooks → your endpoint**.
2. Click **"Add secret"** — Stripe now supports up to 5 active secrets.
3. Add the new secret to your Railway environment variables as a comma-separated
   `STRIPE_WEBHOOK_SECRETS` value:

   ```
   STRIPE_WEBHOOK_SECRETS=whsec_new,whsec_old
   ```

4. Deploy. Both secrets are now active; the server automatically tries each configured secret in
   order — webhooks signed with either will verify.
5. After confirming the new secret works (check logs for successful verifications), remove the old
   secret from the comma-separated list and delete it from the Stripe Dashboard.
6. Update the primary `STRIPE_WEBHOOK_SECRET` env var to the new value and remove the
   `STRIPE_WEBHOOK_SECRETS` list (or keep a single-element list).

## Chimoney Webhook Secret Rotation

The Chimoney webhook endpoint (`POST /api/payments/chimoney/webhook`) uses HMAC-SHA256 verification
over the raw request body. `CHIMONEY_WEBHOOK_SECRET` is required — the server throws at boot in
production if the variable is absent. The URL-path secret variant has been removed.

1. Generate a new secret (via Chimoney console or by request).
2. Update `CHIMONEY_WEBHOOK_SECRET` in Railway (or your secret manager).
3. Update the Chimoney dashboard webhook configuration with the same secret.
4. Deploy. Monitor logs at `POST /api/payments/chimoney/webhook` for successful HMAC verifications.
5. Replay a known recent webhook payload (with the raw body and correct HMAC-SHA256 signature
   computed from the new secret) to confirm the new secret is accepted.
6. Revoke the old secret in the Chimoney console after confirming the new one works.

## OAuth Bridge Secret Rotation

`OAUTH_BRIDGE_SECRET` signs the one-shot HMAC bridge codes used by the OAuth signup flow
(`/auth/oauth-signup`). Codes are short-lived (`OAUTH_BRIDGE_TTL_SECONDS`, default `300`) and
single-use; they replace the previous pattern of passing `providerId`/`email`/`name` in URL query
parameters.

1. Generate a new secret:

   ```bash
   openssl rand -hex 32
   ```

2. Update `OAUTH_BRIDGE_SECRET` in Railway (or your secret manager).
3. Deploy. New OAuth signup flows will immediately use the new signing key.
4. There is no grace-period dual-secret support for bridge codes — codes signed with the old secret
   expire within `OAUTH_BRIDGE_TTL_SECONDS` seconds. Any in-flight OAuth signup initiated just
   before the deploy will fail and the user will need to restart the OAuth flow.
5. Remove the old secret value from all config stores after confirming the new deployment is
   healthy.

## CSRF Secret Rotation

CSRF tokens are signed with a dedicated `CSRF_SECRET` (falls back to `SESSION_SECRET` if
`CSRF_SECRET` is unset — useful for development/staging environments that have not yet added
the variable).

**Rotation order (important — do not reverse):**

1. Generate a new CSRF secret:

   ```bash
   node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
   ```

2. Update `CSRF_SECRET` in Railway (or your secret manager) and deploy.
3. All outstanding CSRF tokens are immediately invalidated — users will receive a 403 on their
   next mutating request and must refresh the page to obtain a new token. User sessions are
   **not** affected; no one is logged out.
4. On a separate deploy cycle, rotate `SESSION_SECRET`. This invalidates active sessions
   (users must log in again) but does not affect CSRF tokens, since they already use the
   new `CSRF_SECRET`.

Keeping the two rotations on different deploy cycles means users experience at most one
disruption at a time — never both simultaneously.
