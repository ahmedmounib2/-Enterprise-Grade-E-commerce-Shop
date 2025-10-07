# Incident Response Playbook

This document explains how we coordinate, communicate, and document responses to production issues
that affect the AhmedMonib E-Shop stack (frontend, backend API, and Expo mobile app). Keep it in the
repository so every engineer, on-call responder, or contractor can reference a single,
version-controlled source of truth.

## When to use this playbook

Use these steps whenever a production-facing surface is degraded or data at risk. Typical triggers
include:

- Uptime or performance alerts (Vercel, Railway, Sentry, Datadog, status checks) breach their
  thresholds.
- Checkout, authentication, or payment flows fail for multiple users.
- Security incidents (suspected compromise, leaked secrets, unusual access patterns).
- Data integrity problems (orders stuck in "processing", duplicate refunds, inventory drift).

If you are unsure whether an issue qualifies, **assume it does** and begin the triage
process—escalations can always be downgraded later.

## Roles & responsibilities

| Role                    | Responsibilities                                                                                                                                 | Primary contact                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------- |
| **Incident commander**  | Owns the response. Coordinates responders, keeps timeline, makes priority calls, ensures communication cadence is met.                           | On-call engineer (see `docs/security/on-call.md`).      |
| **Technical lead**      | Leads debugging/mitigation for the affected surface (frontend, backend, mobile). Can be the same person as commander for low-severity incidents. | Most recent code owner for the impacted area.           |
| **Communications lead** | Publishes customer/internal updates, status page notes, and handoffs to support.                                                                 | Product/Support lead (see `docs/security/contacts.md`). |

Document the assigned roles in the incident log as soon as responders assemble.

## Severity levels

| Level | Description                                                                  | Example signals                                               |
| ----- | ---------------------------------------------------------------------------- | ------------------------------------------------------------- |
| SEV-1 | Full outage or critical data/security breach. Majority of customers blocked. | Stripe webhook queue stalled, checkout unavailable globally.  |
| SEV-2 | Major functionality impaired for a subset of users, workaround available.    | Orders fail for one payment method, admin dashboard unusable. |
| SEV-3 | Degraded experience with limited scope or no customer impact yet.            | Elevated latency, background job retries climbing.            |

Escalate to the next higher severity if impact grows or information is incomplete.

## Response checklist

1. **Acknowledge alert** in the monitoring system and notify the on-call channel (`#incidents`).
2. **Assign roles** (commander, technical lead, comms lead). Start the incident log in
   `docs/INCIDENT_LOG.md` (create entry if absent).
3. **Stabilize**: mitigate customer impact first (feature flag, rollback, queue drain, rate limit,
   etc.). Document actions in real time.
4. **Diagnose**: collect logs, metrics, recent deploy notes. Confirm or disprove hypotheses
   quickly—capture findings in the log.
5. **Communicate**:
   - Internal updates at least every 30 minutes while actively responding.
   - Customer status page updates for SEV-1/SEV-2 incidents every 60 minutes.
   - Open a support ticket template if customer outreach is required.
6. **Resolution**: once impact is mitigated and systems stable, mark the incident resolved and send
   a final update summarizing the fix and next steps.
7. **Post-incident review**: schedule a retrospective within 3 business days. Capture action items
   in Jira and link to the incident log entry.

## Tooling & references

- **Monitoring**: Vercel analytics, Railway metrics, Sentry alerts, Stripe dashboard, Datadog
  synthetic checks.
- **Logs**: Railway application logs, Sentry issues, `backend/logs/` structured logs, Expo error
  logs (see `mobile/dev-setup.md`).
- **Deploys**: `docs/deployment.md`, GitHub Actions history, Vercel/Railway release dashboards.
- **Access & secrets**: `docs/security/access-controls.md` for credential rotation procedures.

## After-action expectations

Each incident must have:

- A completed timeline with owners, actions, and timestamps stored alongside the incident log entry.
- Documented root cause(s) and contributing factors.
- Preventative action items with assignees and due dates tracked in the backlog.
- Communication recap sent to stakeholders (support, product, leadership).

This file should remain in version control so that updates to processes are reviewed just like code.
When procedures change, send a PR that updates both this playbook and any referenced checklists.
