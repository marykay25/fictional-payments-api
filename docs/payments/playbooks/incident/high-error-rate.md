# Fictional Payments API - High Error Rate Incident Playbook

## Purpose

Use this playbook when the Fictional Payments API is experiencing elevated error rates, timeouts, or customer reports of widespread failures.

Examples:

- Error rate above 5% for sustained periods
- Multiple customers experiencing failed transactions
- 5xx responses or timeouts across transaction endpoints
- Alerting rules firing for payments-api health

---

## Symptoms

List what CloudOps or Engineering may observe.

- `payments-api-error-rate-critical` alert or similar
- Increased p95/p99 latency and timeouts
- Customer reports of failed payments
- Log traces showing `PaymentProcessingException` or gateway timeouts

---

## Step-by-Step Investigation

### Step 1 — Confirm the scope

What to do:

1. Open the payments API dashboard: `https://metrics.internal.fictional-payments.io/payments-api`.
2. Determine whether the issue is customer-specific, regional, or service-wide.
3. Record the scope and affected resources in the incident ticket.

Expected result:

- Scope identified (customer, region, environment, or full service).

If this fails:

- Escalate to the Payments Platform Team with the ticket link.

---

### Step 2 — Check service health

What to do:

1. Review availability, error rate, latency, and throughput metrics.
2. Check recent pod restarts and deployment status: `kubectl rollout status deployment/payments-api -n payments-api`.
3. Validate whether alerts correspond to real traffic impact.

Expected result:

- Determine if the service itself is the primary source of the issue.

If this fails:

- Capture logs and metrics snapshots for later analysis.

---

### Step 3 — Check dependencies

What to do:

1. Verify PostgreSQL connectivity and connection pool utilization.
2. Check Redis health and eviction rates.
3. Inspect Kafka consumer lag for settlement and webhook topics.
4. Check external payment processor status pages and recent incidents.

Expected result:

- Identify whether an upstream dependency is degraded.

If this fails:

- Escalate to the dependency owner listed in the Service Overview.

---

### Step 4 — Check recent changes

What to do:

1. Review recent deployments and configuration changes.
2. Check for recent infrastructure changes or auto-scaling events.
3. Link any suspect change to the incident ticket.

Expected result:

- Recent changes are ruled in/out as potential cause.

If this fails:

- Continue investigation and note unknown change status.

---

### Step 5 — Apply approved mitigation

Only perform approved actions listed below.

Approved actions:

1. Scale the API deployment if CPU/memory metrics allow: `kubectl scale deployment payments-api -n payments-api --replicas=<n>`.
2. Toggle circuit breaker feature flag: `PAYMENTS_API_CIRCUIT_BREAKER_ENABLED=true` via ConfigMap or feature flag system.
3. Increase cache TTL temporarily to reduce DB load.
4. If a recent deployment is implicated, perform a controlled rollback: `kubectl rollout undo deployment/payments-api -n payments-api`.

Do not perform:

- Unsafe production changes without approval (e.g., deleting production data).
- Disabling monitoring or alerts.

Expected result:

- Service begins to recover; alert severity reduces.

If this fails:

- Escalate according to the Escalation section below.

---

## Validation

Confirm recovery by checking:

- [ ] Alert cleared or severity reduced.
- [ ] Error rate below target (e.g., < 2%).
- [ ] P99 latency returned to baseline.
- [ ] Ticket updated with timeline and actions.

Validation evidence:

```text
Paste dashboard, log, or ticket evidence here.
```

---

## Escalation

Escalate when:

- Service remains degraded despite approved mitigations.
- The root cause is unknown after standard checks.
- Required access or approvals are missing.

Escalation path:

| Situation | Escalate to | Method |
|---|---|---|
| Service outage | SRE On-Call / Payments Platform Team | PagerDuty / Slack |
| Dependency failure | Dependency owner | Jira / Teams |
| Security concern | Security Operations | Security incident process |

Include this information when escalating:

- Environment
- Affected customers (if applicable)
- Ticket / incident link
- Alert name and time window
- Steps already taken

---

## Related Links

- Service Overview: [Service Overview](../service-overview/service-overview.md)
- Dashboard / Workbook: https://metrics.internal.fictional-payments.io/payments-api
- Logs: https://logs.internal.fictional-payments.io/payments-api
- Alerts: https://alerts.internal.fictional-payments.io/payments-api
- Related operations playbook: ../operations/add-new-payment-type.md
- Repository: https://github.com/fictional-platform/fictional-payments-api


