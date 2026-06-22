---
title: Fictional Payments API - High Error Rate
owner: Platform Reliability Team
severity: P1
last_reviewed: 2026-06-22
automation_available: true
---

# High Error Rate Playbook - Fictional Payments API

## When to use this playbook

Use this playbook when the Fictional Payments API is experiencing error rates exceeding 5% for more than 2 consecutive minutes. This includes:
- Transaction processing failures (payment creation, authorization, settlement)
- Webhook delivery failures
- API endpoint timeouts or 5xx responses
- Rate limiting anomalies affecting legitimate traffic

**Do NOT use this playbook for:**
- Planned maintenance windows (use Maintenance Playbook instead)
- Individual customer issues (escalate to Customer Support)
- Authentication/authorization failures (use Security Incident Playbook)

## Symptoms

- Error rate dashboard shows spike above 5% threshold
- Alert notification: `payments-api-error-rate-critical`
- Customer reports of failed transactions in support channel
- Increased latency on transaction endpoints (p99 > 5000ms)
- Elevated logs showing `PaymentProcessingException` or `GatewayTimeoutError`
- PagerDuty incident automatically triggered
- Canary deployments failing health checks

## Initial checks

### Step 1: Verify alert is legitimate
- Check dashboard at `metrics.internal.fictional-payments.io/payments-api`
- Confirm error rate > 5% in the last 5 minutes
- Validate alert is not due to dashboard lag (check 2-3 refresh cycles)

### Step 2: Identify scope
- Run: `kubectl get pods -n payments-api | grep -i error`
- Check which regions affected: `us-east-1`, `us-west-2`, `eu-west-1`, `ap-southeast-1`
- Identify affected transaction types:
  - All transactions or specific payment methods (ACH, Credit Card, Bank Transfer)?
  - Specific customer segments?
  - Webhook deliveries only?

### Step 3: Check dependencies
```
Dependency Status Check (use internal status page):
- Gateway Service: /health
- Database (RDS Payments DB): Connection pool status
- Message Queue (Kafka): Consumer lag on transaction topics
- Cache Layer (Redis): Memory usage and eviction rate
- External Payment Processor: Their status page
```

### Step 4: Review recent changes
- Check deployment history: `kubectl rollout history deployment/payments-api -n payments-api`
- Review last 3 deployments - were any rolled out in the last 15 minutes?
- Check infrastructure changes in CloudTrail

## Approved actions

### Priority 1: Immediate mitigation (execute in order)

1. **Scale up API pods** (if CPU/Memory is healthy)
   ```
   kubectl scale deployment payments-api -n payments-api --replicas=10
   (Current: usually 5-6, max safe: 12)
   ```

2. **Enable circuit breaker** to prevent cascading failures
   - Feature flag: `PAYMENTS_API_CIRCUIT_BREAKER_ENABLED=true`
   - Update via ConfigMap: `kubectl edit configmap payments-api-config`
   - Restart pods: `kubectl rollout restart deployment/payments-api -n payments-api`

3. **Increase cache TTL** temporarily to reduce database load
   ```
   Update Redis config:
   TRANSACTION_CACHE_TTL: 300s (from 60s)
   STATUS_CACHE_TTL: 120s (from 30s)
   ```

4. **Drain unhealthy pods** if error rate is concentrated
   - Identify pod: `kubectl logs <pod-name> -n payments-api | grep ERROR | head -20`
   - Cordon and drain: `kubectl cordon <node-name>` then `kubectl drain <node-name>`

### Priority 2: Investigate root cause

5. **Check database connection pool**
   ```sql
   SELECT count(*) as connection_count FROM pg_stat_activity 
   WHERE datname = 'payments_prod';
   ```
   - If > 80% of max_connections (200), scale up or investigate leaks

6. **Review error logs for patterns**
   ```
   kubectl logs -n payments-api -l app=payments-api --since=5m | grep ERROR | jq '.message' | sort | uniq -c | sort -rn
   ```
   - Look for repeated error messages
   - Check error rates by endpoint: `/api/v1/transactions`, `/api/v1/webhooks`, etc.

7. **Check message queue consumer lag**
   ```
   Kafka Consumer Lag Command:
   kafka-consumer-groups --bootstrap-server kafka-broker:9092 --group payments-settlement-group --describe
   ```
   - Lag > 10,000 messages indicates processing backlog

### Priority 3: Stabilize and recover

8. **If database is the bottleneck:**
   - Reduce query verbosity: Disable transaction logging temporarily
   - Kill long-running queries: `SELECT pg_terminate_backend(pid) WHERE query_start < NOW() - interval '30 seconds';`

9. **If external gateway is slow:**
   - Implement timeout fallback to queue transactions for retry
   - Feature flag: `ENABLE_PAYMENT_GATEWAY_FALLBACK=true`

10. **Rollback if recent deployment is identified as cause:**
    ```
    kubectl rollout undo deployment/payments-api -n payments-api
    kubectl rollout status deployment/payments-api -n payments-api
    ```

## Do NOT actions

❌ **DO NOT** manually delete pods or force restart entire deployment
- This will cause brief service interruption; coordinate with on-call if required

❌ **DO NOT** immediately increase timeouts
- This masks underlying issue and makes problem worse

❌ **DO NOT** disable monitoring or alerts
- We need continuous visibility during incident

❌ **DO NOT** modify production database directly
- Use managed queries through DBA on-call
- Never drop indexes or disable constraints

❌ **DO NOT** clear entire cache layer
- Do targeted cache invalidation only for problematic keys

❌ **DO NOT** redirect traffic to backup region manually
- Failover is automated; manual changes cause state inconsistencies

❌ **DO NOT** contact external payment processor support immediately
- First confirm it's not our infrastructure; they will confirm status on their end

## Validation

### Success criteria
- [ ] Error rate drops below 2% within 5 minutes of mitigation
- [ ] P99 latency returns to baseline (< 1500ms)
- [ ] No increase in failed transaction count after 10 minutes
- [ ] Customer complaints cease in support channels
- [ ] Database connection pool returns to normal (< 60% utilization)

### Automated validation
```bash
# Run validation checks
./scripts/validate_payments_health.sh

# Expected output:
# ✓ Error rate: 1.2% (PASS - under 5% threshold)
# ✓ Latency P99: 823ms (PASS - under 1500ms)
# ✓ Database connections: 92/200 (PASS - under 80%)
# ✓ Queue lag: 245 messages (PASS - under 10k)
```

## Escalation

### Escalation path
- **Tier 1 (SRE On-Call):** Implement initial checks and Priority 1 mitigation actions
  - Duration: 0-5 minutes
  - Contact: Page on-call SRE via PagerDuty

- **Tier 2 (Payments Platform Lead):** If error rate not reduced after 5 min
  - Duration: 5-15 minutes
  - Responsibilities: Deep database/infrastructure investigation
  - Contact: `@payments-platform-leads` Slack

- **Tier 3 (VP Engineering):** If service remains degraded after 15 minutes
  - Duration: 15+ minutes
  - Decision: Customer communication, failover, external escalation
  - Contact: Page VP Engineering via PagerDuty
  - Action: Initiate incident war room in Zoom link: https://fictional-payments.zoom.us/warroom

### External escalation
- **External Payment Processor:** Only escalate after confirming our infrastructure is healthy
- **Customers:** VP Product notifies high-value customers if estimated recovery > 30 min
- **Status Page:** Post updates every 5 minutes if incident lasts > 10 minutes

## References

- **Dashboard:** [Payments API Metrics](https://metrics.internal.fictional-payments.io/payments-api)
- **Runbook:** [Kubernetes Troubleshooting Guide](https://internal-wiki.fictional-payments.io/runbooks/k8s-debug)
- **Alerting:** [Alert Configuration](https://prometheus.internal.fictional-payments.io/alerts?search=payments)
- **On-Call Schedule:** [PagerDuty - Payments Team](https://fictional-payments.pagerduty.com/schedules/payments-api)
- **Postmortem Template:** [Incident Postmortem](https://internal-wiki.fictional-payments.io/postmortems/template)
- **Related Playbooks:**
  - [Database Performance Degradation](./database-performance.md)
  - [External Gateway Failures](./gateway-failures.md)
  - [Security Incident Response](./security-incident.md)
- **Slack Channels:** `#payments-alerts`, `#payments-incidents`, `#oncall`
- **War Room Setup:** [Incident War Room Zoom](https://fictional-payments.zoom.us/warroom)
