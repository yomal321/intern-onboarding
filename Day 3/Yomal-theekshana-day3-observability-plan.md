# BookSwap — Observability Plan

---

## Setup

### Logs — Azure Monitor Logs

**Schema:** Every log entry emitted by the API follows this structure:

```json
{
  "timestamp": "2026-05-22T08:30:00.000Z",
  "level": "info",
  "event": "loan.created",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "memberId": "member-1",
  "bookId": "book-1",
  "loanId": "loan-1"
}
```

**Auditable events logged (required by NFR):**

| Event | Fields logged |
|-------|---------------|
| `auth.failed` | `timestamp`, `requestId`, `memberId` (from JWT if parseable, else `anonymous`), `reason` |
| `loan.created` | `timestamp`, `requestId`, `memberId`, `bookId`, `loanId`, `dueDate` |
| `loan.returned` | `timestamp`, `requestId`, `memberId`, `bookId`, `loanId`, `returnedAt` |
| `book.listed` | `timestamp`, `requestId`, `memberId`, `bookId` |
| `borrow.requested` | `timestamp`, `requestId`, `memberId`, `bookId`, `requestId` |

**Fields never logged (PII redaction rules):**

- `email` — always redacted, replaced with `[REDACTED]`
- `displayName` — never logged
- `address` — never stored or logged at any layer
- `phone` — never stored or logged at any layer
- JWT token raw value — never logged even on auth failure; log only the `sub` claim and the failure reason
- `search` query parameter value — redacted before telemetry leaves the process (may contain resident names)

**Redaction middleware (applied before any log leaves the process):**

```javascript
const appInsights = require('applicationinsights');

appInsights.defaultClient.addTelemetryProcessor((envelope) => {
  const props = envelope.data?.baseData?.properties || {};
  if (props.email)        props.email = '[REDACTED]';
  if (props.displayName)  props.displayName = '[REDACTED]';

  // Redact search query param from request URLs
  const url = envelope.data?.baseData?.url || '';
  envelope.data.baseData.url = url.replace(
    /([?&]search=)[^&]*/gi, '$1[REDACTED]'
  );
  return true;
});
```

**Retention:** 90 days in Azure Monitor Logs (hot tier). After 90 days, archive to Azure Blob Storage cold tier for 2 years to satisfy compliance requirements.

---

### Metrics — Azure Application Insights

Instrumented via the Application Insights Node.js SDK.

**Auto-collected metrics:**

| Metric | Description |
|--------|-------------|
| `requests` | Every HTTP request — name, duration, resultCode, success |
| `dependencies` | Every SQL query and Redis command — type, duration, success |
| `exceptions` | Every unhandled error — outerMessage, stack (sanitised) |

**Custom metrics emitted by the API:**

| Metric name | Type | Emitted on |
|-------------|------|------------|
| `bookswap.listing.created` | Counter | Successful `POST /books` → 201 |
| `bookswap.borrow.requested` | Counter | Successful `POST /books/{id}/borrow-requests` → 201 |
| `bookswap.loan.created` | Counter | Borrow request accepted → Loan record inserted |
| `bookswap.loan.returned` | Counter | Successful `PATCH /loans/{id}` → 200 |
| `bookswap.auth.failed` | Counter | Any 401 response; dimension: `reason` (expired / missing / invalid) |
| `bookswap.cache.fallback` | Counter | Redis unavailable, fell back to SQL for a search request |

---

### Traces — Application Insights Distributed Tracing

**Sample rate:**
- First 30 days: **100%** — low traffic, need full visibility to establish baselines
- After baseline: **10%** for successful requests, **100%** for all errors and requests exceeding 800 ms

**Trace spans captured per request:**

1. Incoming HTTP request — Front Door → App Service (includes `x-request-id` header injected by Front Door)
2. JWT validation — duration of Entra JWKS fetch and token verification
3. Redis GET — cache hit or miss, duration
4. SQL query — parameterised query text (no bind values), duration, row count
5. Redis SET — cache populate, TTL set, duration
6. Service Bus publish — message enqueue duration (fire-and-forget; span marked async)

**Correlation:** Azure Front Door injects an `x-request-id` header on every inbound request. This ID is propagated through all spans as `operation_Id` and appears in every log entry as `requestId`, making it possible to reconstruct the full trace for any logged event.

---

## Signals Table

| # | Signal type | Source | What it answers | Sample query / metric name |
|---|-------------|--------|-----------------|---------------------------|
| 1 | Metric | Application Insights `requests` | Search latency p95 — are we meeting the 800 ms SLO? | `requests \| where name == "GET /books" \| summarize percentile(duration, 95) by bin(timestamp, 5m)` |
| 2 | Metric | Application Insights `requests` | Listing creation success rate — are we meeting 99.9% SLO? | `requests \| where name == "POST /books" \| summarize good = countif(toint(resultCode) < 500), total = count() \| extend rate = 100.0 * good / total` |
| 3 | Log | Application Insights `traces` | Auth failures with member ID — detect brute force or token issues | `traces \| where customDimensions.event == "auth.failed" \| project timestamp, memberId = customDimensions.memberId, requestId = customDimensions.requestId` |
| 4 | Trace | Application Insights `dependencies` | Slow request breakdown — is SQL or Redis causing the latency? | `dependencies \| where timestamp > ago(1h) \| where type in ("SQL", "Redis") \| summarize avg(duration) by type, bin(timestamp, 5m)` |
| 5 | Metric | Azure Service Bus `ActiveMessages` | Email digest queue depth — is the consumer keeping up? | Azure Monitor metric: `ActiveMessages` on namespace `bookswap-sb` — alert if > 1 000 messages |
| 6 | Metric | Application Insights `requests` | Request rate — detect Sunday traffic spike early | `requests \| summarize rps = count() / 60.0 by bin(timestamp, 1m) \| where rps > 50` |
| 7 | Log | Application Insights `traces` | Loan audit trail — every loan creation and return with member ID | `traces \| where customDimensions.event in ("loan.created", "loan.returned") \| project timestamp, memberId = customDimensions.memberId, loanId = customDimensions.loanId, requestId = customDimensions.requestId` |
| 8 | Metric | Azure Cache for Redis `CacheHitRate` | Cache effectiveness — is Redis actually helping? | Azure Monitor metric: `cachehits / (cachehits + cachemisses)` — alert if < 50% for 10 min |
| 9 | Metric | App Service `CpuPercentage` | App Service health during spike — is autoscale keeping up? | Azure Monitor metric: `CpuPercentage > 70%` on App Service Plan for 5 min |
| 10 | Log | Application Insights `exceptions` | Unhandled errors — catch anything unexpected before users report it | `exceptions \| where timestamp > ago(1h) \| summarize count() by outerMessage \| order by count_ desc` |

---

## Results Summary

| Metric | Target | Achieved |
|--------|--------|----------|
| SLOs covered by an alert | 100% | 100% — all SLOs have at least one alert |
| Alerts with a clear runbook link | 100% | 100% — each alert links to a runbook section |
| Dashboards for ops | 1 health + 1 business | Both defined below |
| PII fields redacted before logging | 100% | 100% — email, displayName, search param, raw JWT |

---

## Alert Proposal

| Alert | Condition | Severity | Notification | Runbook |
|-------|-----------|----------|--------------|---------|
| Search SLO burn | p95 latency on `GET /books` > 800 ms for 5 consecutive minutes | Sev 2 | PagerDuty + Teams `#bookswap-incidents` | `reliability-runbook.md#failure-2-redis` |
| Listing creation failure | 5xx rate on `POST /books` > 0.1% over 5 min | Sev 1 | PagerDuty (wake on-call) + Teams `#bookswap-incidents` | `reliability-runbook.md#failure-1-sql` |
| Complete listings outage | `GET /books` success rate = 0% for 2 consecutive minutes | Sev 1 | PagerDuty + SMS to on-call | `reliability-runbook.md#failure-1-sql` |
| Auth failure spike | `auth.failed` events > 50 per minute for 2 min | Sev 2 | Teams `#bookswap-security` | `security-review.md#authn` |
| Redis cache down | Redis `cachehits` = 0 for 3 consecutive minutes | Sev 2 | Teams `#bookswap-incidents` | `reliability-runbook.md#failure-2-redis` |
| Traffic spike detected | RPS > 5× 28-day median baseline for 3 min | Sev 2 | Teams `#bookswap-ops` | `reliability-runbook.md#failure-3-spike` |
| SQL dependency failing | SQL dependency failure rate > 50% over 2 min | Sev 1 | PagerDuty | `reliability-runbook.md#failure-1-sql` |
| Service Bus queue depth | `ActiveMessages` > 1 000 for 5 min | Sev 3 | Teams `#bookswap-ops` (no page) | `reliability-runbook.md#failure-3-spike` |
| Audit log gap | No `loan.created` or `loan.returned` events for 24 h when active loans exist | Sev 2 | Teams `#bookswap-security` | `observability-plan.md#audit` |
| Cache fallback rate rising | `bookswap.cache.fallback` > 10 events per minute for 5 min | Sev 2 | Teams `#bookswap-incidents` | `reliability-runbook.md#failure-2-redis` |

---

## alerts.yaml

```yaml
alerts:
  - name: SearchSLOBurn
    description: Search p95 latency exceeds 800 ms SLO
    condition: percentile(GET /books duration, 95) > 800ms over 5m
    severity: 2
    notification:
      - pagerduty
      - teams: "#bookswap-incidents"
    runbook: reliability-runbook.md#failure-2-redis

  - name: ListingCreationFailure
    description: POST /books 5xx rate exceeds error budget
    condition: 5xx_rate(POST /books) > 0.1% over 5m
    severity: 1
    notification:
      - pagerduty
      - teams: "#bookswap-incidents"
    runbook: reliability-runbook.md#failure-1-sql

  - name: CompleteListingsOutage
    description: GET /books returning zero successes for 2 minutes
    condition: success_rate(GET /books) == 0% for 2m
    severity: 1
    notification:
      - pagerduty
      - sms: on-call
    runbook: reliability-runbook.md#failure-1-sql

  - name: AuthFailureSpike
    description: Possible brute force or token misconfiguration
    condition: count(auth.failed) > 50/min for 2m
    severity: 2
    notification:
      - teams: "#bookswap-security"
    runbook: security-review.md#authn

  - name: RedisCacheDown
    description: Redis hit rate dropped to zero — all reads hitting SQL
    condition: cachehits == 0 for 3m
    severity: 2
    notification:
      - teams: "#bookswap-incidents"
    runbook: reliability-runbook.md#failure-2-redis

  - name: TrafficSpike
    description: Request rate 5× above 28-day median baseline
    condition: rps > 5x_28d_baseline for 3m
    severity: 2
    notification:
      - teams: "#bookswap-ops"
    runbook: reliability-runbook.md#failure-3-spike

  - name: SQLDependencyFailing
    description: More than half of SQL calls failing
    condition: sql_failure_rate > 50% over 2m
    severity: 1
    notification:
      - pagerduty
    runbook: reliability-runbook.md#failure-1-sql

  - name: ServiceBusQueueDepth
    description: Email digest consumer falling behind spike load
    condition: ActiveMessages > 1000 for 5m
    severity: 3
    notification:
      - teams: "#bookswap-ops"
    runbook: reliability-runbook.md#failure-3-spike

  - name: AuditLogGap
    description: Loan events absent despite active loans — logging pipeline may be broken
    condition: no loan.created or loan.returned events in 24h when loans table has active records
    severity: 2
    notification:
      - teams: "#bookswap-security"
    runbook: observability-plan.md#audit

  - name: CacheFallbackRising
    description: API falling back to SQL for search at elevated rate
    condition: bookswap.cache.fallback > 10/min for 5m
    severity: 2
    notification:
      - teams: "#bookswap-incidents"
    runbook: reliability-runbook.md#failure-2-redis
```

---

## Dashboards

### Health Dashboard (ops team — real time)

| Panel | Signal | Alert threshold shown |
|-------|--------|-----------------------|
| Request rate (RPS) | App Insights `requests` count per minute | Red line at 5× baseline |
| Search p95 latency | App Insights `percentile(duration, 95)` on `GET /books` | Red line at 800 ms |
| Listing creation success rate | App Insights 5xx rate on `POST /books` | Red line at 0.1% failure |
| SQL dependency health | App Insights `dependencies` — SQL failure rate | Red line at 10% |
| Redis hit rate | Azure Monitor `cachehits / (cachehits + cachemisses)` | Red line at 50% |
| Active App Service instances | App Service instance count | Shows autoscale activity |
| Service Bus queue depth | Azure Monitor `ActiveMessages` | Red line at 1 000 |

### Business Dashboard (product team — daily)

| Panel | Signal |
|-------|--------|
| Books listed today | COUNT of `bookswap.listing.created` custom metric |
| Borrow requests today | COUNT of `bookswap.borrow.requested` |
| Loans active right now | COUNT of `bookswap.loan.created` minus `bookswap.loan.returned` |
| Loans returned today | COUNT of `bookswap.loan.returned` |
| Top 10 most borrowed books | JOIN of `loan.created` events grouped by `bookId` |
| Auth failure rate (7-day trend) | COUNT of `bookswap.auth.failed` grouped by `reason` dimension |

---

## What We Are Deliberately NOT Alerting On

1. **Individual slow requests (single outliers).** One 900 ms response does not breach the SLO. Alerting on every slow request creates noise that trains the team to ignore pages. We alert only on sustained p95 degradation over 5 consecutive minutes.

2. **Redis cache misses on cold start.** Every deployment causes a brief period of cache misses while keys repopulate from SQL. This is expected and transient — alerting on it would fire on every release and numb the team to the Redis-down alert that actually matters.

3. **Service Bus queue depth below 1 000 messages.** The queue is designed to absorb listing bursts. A depth of 200–800 is normal during active listing periods on Sunday evenings. Alerting too early creates noise without actionable signal; the consumer will catch up when the spike subsides.

4. **4xx responses on write endpoints.** `400 Bad Request` and `422 Unprocessable Entity` are correct API behaviour — they mean the client sent invalid input. Alerting on 4xx would fire on every integration test run and every badly-formed mobile app request. We monitor 5xx only.

5. **Individual auth failures (single 401s).** One 401 means a member's token expired — completely normal behaviour. We alert only when auth failures spike above 50 per minute sustained over 2 minutes, which indicates a systemic misconfiguration or a credential-stuffing attack.

6. **p50 (median) latency.** Median latency is almost always well within SLO and does not represent the worst-case member experience. We track p95 only, which captures the slowest 5% of requests — the members most likely to abandon the app.
