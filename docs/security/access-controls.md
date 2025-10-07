# Access Control Policy

This guide documents how we grant, audit, and revoke access across the AhmedMonib E-Shop stack so we
stay compliant with least-privilege practices.

---

## Account provisioning

1. **Request**: Manager or team lead opens an access ticket specifying systems required (GitHub,
   Vercel, Railway, Stripe, Expo, databases).
2. **Approval**: Security lead or engineering manager approves the request in writing.
3. **Provision**:
   - Add the user to the appropriate GitHub team (`eshop-core`, `eshop-readonly`, etc.).
   - Invite the user to Vercel and Railway with the minimum role needed (usually _Developer_).
   - Issue SSO access to third-party tools (Stripe, Datadog, Sentry) through the identity provider.
4. **Secrets**: Never share `.env` files directly. Instead, grant read access to the password
   manager or secret vault entry (`E-Shop/Prod/Backend`, etc.).
5. **Verification**: The requester confirms the user can access required systems and nothing more.

Document every grant in the access ticket so we can trace who approved and provisioned each account.

---

## Role definitions

| Role              | Permissions                                                                                    |
| ----------------- | ---------------------------------------------------------------------------------------------- |
| **Admin**         | Full control (manage billing, secrets, infrastructure). Restricted to two senior engineers.    |
| **Developer**     | Deploy code, view logs, manage feature flags, read secrets needed for day-to-day operations.   |
| **Read-only**     | View dashboards/logs without modifying resources. Ideal for contractors or auditors.           |
| **Support agent** | Limited access to support tooling (e.g., Stripe refunds) with no deploy or secret permissions. |

Assign the lowest role that unlocks the tasks a person must perform.

---

## Onboarding checklist

- [ ] Ticket created and approved.
- [ ] User added to GitHub team(s) and 2FA enforced.
- [ ] Vercel, Railway, and Expo invites sent with developer role.
- [ ] Stripe/Sentry/Datadog access granted via SSO group.
- [ ] Secrets vault access provisioned with read-only scope.
- [ ] User acknowledged the security handbook and incident response process.
- [ ] Ticket updated with date/time of completion and stored in the security archive.

---

## Offboarding checklist

Perform within 4 hours of notification:

- [ ] Revoke SSO access and rotate shared credentials (Stripe restricted keys, Railway tokens).
- [ ] Remove from GitHub teams and Vercel/Railway projects.
- [ ] Invalidate API tokens or CLI auth sessions (Vercel, Expo, Datadog).
- [ ] Rotate secrets the user could have accessed if compromise suspected.
- [ ] Disable device certificates for VPN or SSH bastion hosts.
- [ ] Update the offboarding ticket with confirmation and responsible engineer.

---

## Periodic reviews

- **Monthly**: Review Vercel/Railway member lists and downgrade or remove unused accounts.
- **Quarterly**: Audit GitHub teams against the HR roster. Remove contractors whose engagements
  ended.
- **Semi-annual**: Rotate long-lived credentials (Stripe restricted keys, Database passwords) and
  ensure service accounts still serve a valid purpose.

Document review findings in `docs/security/access-review-logs/` (create per-month markdown files).

---

## Emergency access

If production access is urgently required outside normal procedures:

1. Incident commander approves temporary elevation in the incident ticket.
2. Security lead grants access with an expiration time (12 hours maximum).
3. After the incident, revert the account to its prior role and note the change in the incident log.

Keep emergency elevation rare and well documented to maintain compliance.
