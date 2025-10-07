# On-call Roster & Expectations

This document outlines how we staff the incident on-call rotation referenced in the incident
response playbook.

## Rotation

- Primary on-call rotates weekly every Monday at 09:00 UTC.
- Secondary on-call mirrors the primary schedule and provides backup coverage.
- Update the current roster in `docs/security/on-call-roster.csv` (create if missing) whenever the
  schedule changes.

## Responsibilities

- Acknowledge alerts in under 5 minutes during coverage hours.
- Coordinate with the incident commander role described in `docs/INCIDENT_RESPONSE.md`.
- Ensure handoff notes are posted in the `#on-call-handoff` channel at the end of each shift,
  including active incidents, flaky alerts, and pending follow-ups.

## Handoff checklist

- [ ] Confirm pager/notification device is operational.
- [ ] Review incidents from the previous week and outstanding action items.
- [ ] Validate access to dashboards (Vercel, Railway, Datadog, Sentry, Stripe).
- [ ] Sync with product/support on any customer-impacting issues.

## Escalation path

1. If the primary does not acknowledge within 5 minutes, the alert pages the secondary.
2. If both primary and secondary fail to respond within 10 minutes, escalate to the engineering
   manager via phone.
3. For critical (SEV-1) incidents, notify the CTO after mitigation steps begin.

Keep the roster current and accessible so every responder knows who is covering and how to reach
them.
