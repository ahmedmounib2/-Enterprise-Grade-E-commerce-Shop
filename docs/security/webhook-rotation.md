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

4. **TODO:** Update the Stripe webhook signature verification code in
   `backend/src/controllers/payment.controller.js` (`stripeWebhook` function) to iterate over the
   comma-separated secrets and try each one when verifying signatures. Currently only a single
   `STRIPE_WEBHOOK_SECRET` is read; multi-secret support is needed for dual-secret rotation.
5. Deploy. Both secrets are now active and incoming webhooks signed with either will verify.
6. After confirming the new secret works (check logs for successful verifications), remove the old
   secret from the comma-separated list and delete it from the Stripe Dashboard.
7. Update the primary `STRIPE_WEBHOOK_SECRET` env var to the new value and remove the
   `STRIPE_WEBHOOK_SECRETS` list (or keep a single-element list).

## Chimoney Webhook Secret Rotation

The Chimoney webhook endpoint (`POST /api/chimoney/webhook`) uses HMAC-SHA256 verification over the
raw request body. `CHIMONEY_WEBHOOK_SECRET` is required — the server throws at boot in production if
the variable is absent. The URL-path secret variant has been removed.

1. Generate a new secret (via Chimoney console or by request).
2. Update `CHIMONEY_WEBHOOK_SECRET` in Railway (or your secret manager).
3. Update the Chimoney dashboard webhook configuration with the same secret.
4. Deploy. Monitor logs at `POST /api/chimoney/webhook` for successful HMAC verifications.
5. Replay a known recent webhook payload (with the raw body and correct HMAC-SHA256 signature
   computed from the new secret) to confirm the new secret is accepted.
6. Revoke the old secret in the Chimoney console after confirming the new one works.
