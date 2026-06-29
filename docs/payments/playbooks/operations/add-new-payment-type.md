# Add New Payment Type - Operations Playbook

## Purpose

Use this playbook to add and enable a new payment type (for example, a new card network or wallet) in the Fictional Payments API.

Do not use this playbook for:

- Making schema changes to the primary transaction store without a deployment plan
- Performing ad-hoc data migrations in production without approval

---

## Preconditions

Before starting, confirm:

- [ ] Change is approved and ticket exists
- [ ] Required approvals from product and compliance (if applicable)
- [ ] You have required access to deployment pipelines and feature flag service
- [ ] No active incident impacting the payments service

---

## Inputs Needed

| Input | Example | Required |
|---|---|---|
| Ticket / request link | JIRA-1234 | Yes |
| Environment | Staging / Production | Yes |
| Payment type identifier | example_wallet_v2 | Yes |
| Processor configuration | Processor API keys / endpoints | Yes |
| Compliance checklist | PCI requirements confirmation | If applicable |

---

## Access Required

Required access:

- Access to the deployment pipeline (CI/CD)
- Access to feature flag management console
- Kubernetes deployment access for `payments-api` namespace
- Secrets manager access for adding processor credentials

If access is missing:

1. Stop the procedure.
2. Update the ticket with the missing access.
3. Escalate to Platform Engineering or Security as appropriate.

---

## Procedure

### Step 1 — Prepare configuration in staging

What to do:

1. Create a configuration entry for the new payment type in staging (feature flag + processor config).
2. Add necessary secrets to the staging secrets manager with restricted scope.
3. Deploy the config to staging via the normal pipeline.

Expected result:

- The staging environment accepts transactions for the new payment type without errors.

If this fails:

- Check application logs and secret access; revert config if it causes errors.

---

### Step 2 — Run end-to-end tests

What to do:

1. Execute smoke tests that cover create, authorize, capture, and webhook flows for the new payment type.
2. Verify reconciliation and settlement events are emitted to Kafka topics.

Expected result:

- All smoke tests pass and event flow is validated.

If this fails:

- Capture failing test output and revert configuration; investigate with Engineering.

---

### Step 3 — Enable in production (controlled rollout)

What to do:

1. Schedule a low-traffic release window if required by compliance.
2. Add the processor credentials to production secrets manager.
3. Enable the feature flag for a small percentage of traffic or specific test customers.
4. Monitor metrics and logs closely for errors or increased latency.

Expected result:

- New payment type receives traffic per rollout plan and behaves as expected.

If this fails:

- Disable the flag and follow rollback steps below.

---

## Validation

Validation checklist:

- [ ] Staging tests passed
- [ ] Production can process test transactions for the new type
- [ ] Webhooks and settlement events are observed in Kafka
- [ ] No increase in error rate or latency beyond thresholds
- [ ] Ticket updated with evidence and logs

Validation evidence:

```text
Paste test output, dashboard links, or log snippets here.
```

---

## Rollback / Backout

Rollback steps:

1. Disable the feature flag immediately.
2. Remove processor credentials from production secrets manager.
3. Revert any configuration changes via the deployment pipeline.
4. Notify stakeholders and open a follow-up investigation ticket.

If rollback is not possible:

- Escalate to Platform Engineering and the Payments Product Owner with risk details.

---

## Communication

Update the ticket with:

- What was completed
- When it was completed
- Who completed it
- Validation evidence

If requester notification is required, use the standard success message format in the ticket.

---

## Escalation

Escalate when:

- Required access or approvals are missing
- The procedure fails or causes regressions
- The change may impact production customers beyond the rollout plan

Escalation path:

| Situation | Escalate to | Method |
|---|---|---|
| Missing access | Platform Engineering | Jira / Teams |
| Procedure failure | Payments Platform Team | PagerDuty / Slack |
| Production impact | Incident Commander | Incident process |

---

## Related Links

- Service Overview: ../service-overview/service-overview.md
- Related incident playbook: ../incident/high-error-rate.md
- Deployment pipeline: https://ci.internal.fictional-payments.io
