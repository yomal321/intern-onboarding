# BookSwap — Reliability Runbook v0.1
 
---
 
## Failure 1: Azure SQL primary unavailable for 5 minutes
 
### What the user sees
 
- `GET /books` returns `503 Service Unavailable` — search is completely broken
- `POST /books` returns `503` — members cannot list a new book
- `POST /books/{bookId}/borrow-requests` returns `503` — borrow requests cannot be submitted
- `PATCH /borrow-requests/{requestId}` returns `503` — owners cannot accept or decline requests
- `/health` still returns `200` — it must not take a SQL dependency
- Members see: _"Something went wrong. Please try again shortly."_
### Detection
 
Azure Application Insights tracks SQL as a dependency call. If more than 50% of SQL calls fail within a 2-minute window, an alert fires to PagerDuty and posts to the `#bookswap-incidents` Teams channel.
 
```kusto
dependencies
| where timestamp > ago(5m)
| where type == "SQL"
| summarize
    failed = countif(success == false),
    total  = count()
| where failed > 0.5 * total
```
 
Alert condition: query result row exists → Severity 1 → PagerDuty page + Teams `#bookswap-incidents`.
Expected detection time: under 2 minutes.
 
### Mitigation in design (timeouts, retries, circuit breaker, fallback)
 
**Connection pool — fail fast:**
 
```
SET max connections        = 20
SET connection timeout     = 3 seconds
SET idle timeout           = 10 seconds
 
IF a connection is not available within 3 seconds:
  RETURN 503 immediately
  DO NOT queue the request
```
 
**Retry with exponential backoff:**
 
```
SET max attempts = 3
SET base wait    = 200 ms
 
FOR each attempt 1..3:
  TRY the database call
  IF it succeeds → RETURN result
 
  IF error is transient (connection refused / timeout / deadlock):
    IF this was attempt 3 → THROW, return 503
    WAIT (200ms × 2^attempt) + random jitter 0–100 ms
    CONTINUE to next attempt
 
  IF error is non-transient (constraint violation, bad SQL) → THROW immediately, do not retry
```
 
**Circuit breaker:**
 
```
KEEP a counter of consecutive SQL failures
 
IF failures >= 5 within 30 seconds:
  OPEN the circuit for 30 seconds
  RETURN 503 to all requests without touching the database
 
AFTER 30 seconds → HALF-OPEN: allow one probe request
  IF probe succeeds → CLOSE circuit, reset counter
  IF probe fails    → OPEN again for another 30 seconds
```
 
**Idempotency key for listing creation:**
 
```
WHEN POST /books is received:
  READ Idempotency-Key header (UUID v4, client-generated)
 
  IF key exists in idempotency_keys table (TTL 24 hours):
    RETURN the stored 201 response immediately
    DO NOT insert a duplicate book record
 
  IF key is new:
    BEGIN transaction
      INSERT book record
      INSERT idempotency_keys row (key, response_body, expires_at = now + 24h)
    COMMIT
    RETURN 201 Created
```
 
This ensures that a member who retries `POST /books` after a SQL timeout never sees two identical listings appear.
 
**Azure SQL auto-failover group:**
 
```
Connection string points to the failover group listener endpoint, NOT the primary hostname
 
IF primary becomes unavailable:
  Azure promotes the geo-replica automatically
  Failover completes in under 30 seconds
  App reconnects through the same listener endpoint — no config change needed
```
 
### Manual response (who is paged, what they do)
 
1. **On-call engineer** is paged via PagerDuty (Severity 1) — expected acknowledgement within 5 minutes.
2. Open Azure Portal → SQL Server → Overview → confirm server status and active connections.
3. Open Azure Service Health → check for a regional SQL outage.
4. If regional outage confirmed → Azure Portal → SQL Server → Failover groups → **Initiate manual failover** to the secondary region.
5. If server is healthy but pool is exhausted → App Service → Restart the affected instance(s).
6. If restart does not clear the issue → scale App Service plan up one tier to get fresh instances.
7. Post a status update in `#bookswap-status` within 5 minutes of acknowledgement and every 15 minutes until resolved.
### Post-incident actions
 
- [ ] Confirm Azure SQL auto-failover group is configured with a replica in a secondary region
- [ ] Add a `/health/db` sub-check endpoint that tests SQL separately so the main `/health` can stay green during partial degradation
- [ ] Add a read replica so `GET /books` queries can serve from the replica when the primary is under stress
- [ ] Run a chaos test: intentionally block SQL for 60 seconds and verify circuit breaker opens and 503s are returned within the expected window
- [ ] Write a blameless post-mortem within 48 hours and share it in `#bookswap-engineering`
---
 
## Failure 2: Azure Cache for Redis is down
 
### What the user sees
 
- `GET /books` and `GET /books/{bookId}` still return correct results — no errors
- Response times increase from the normal 20–80 ms to 300–700 ms as every request hits Azure SQL directly
- If the Sunday spike and a Redis outage coincide, latency may breach the 800 ms SLO
- Borrow request and loan endpoints are unaffected — they do not use the cache
- Members see no error message; the slowdown is the only symptom
### Detection
 
Azure Monitor tracks Redis cache hits as a native metric. If the hit rate drops to zero and stays there for 3 consecutive minutes, an alert fires.
 
```kusto
AzureMetrics
| where ResourceType == "MICROSOFT.CACHE/REDIS"
| where MetricName == "cachehits"
| summarize hits = sum(Total) by bin(TimeGenerated, 1m)
| where hits == 0
```
 
Alert condition: hit rate = 0 for 3 minutes → Severity 2 → Teams `#bookswap-incidents`.
This is not a PagerDuty page because the system is still serving correct data — just slower.
 
Secondary signal: Application Insights `dependencies` table — Redis dependency `success == false` rate > 80% for 2 minutes.
 
```kusto
dependencies
| where timestamp > ago(5m)
| where type == "Redis"
| summarize
    failed = countif(success == false),
    total  = count()
| extend failure_rate = 100.0 * failed / total
| where failure_rate > 80
```
 
### Mitigation in design
 
**Cache-aside pattern with graceful fallback:**
 
```
WHEN GET /books or GET /books/{bookId} is received:
 
  SET redis_timeout = 500 ms
 
  TRY read from Redis (timeout = 500 ms)
    IF hit  → RETURN cached data, done
    IF miss → continue
    IF Redis unreachable or timeout → LOG warning, continue (do not throw)
 
  READ from Azure SQL
    IF not found → RETURN 404
 
  TRY write result back to Redis with TTL = 60 seconds
    IF Redis write fails → LOG warning, ignore — still RETURN the SQL result
 
RETURN result to caller
```
 
**Redis connection — fail fast:**
 
```
SET connect timeout      = 1 second
SET command timeout      = 500 ms
SET max retries per call = 1
SET retry wait           = 100 ms
 
IF Redis does not respond within 1 second on connect:
  MARK Redis as unavailable
  SKIP all cache reads and writes for the next 10 seconds
  RECHECK Redis after 10 seconds (exponential back-off up to 60 seconds max)
```
 
**What is deliberately not cached:**
- Borrow request status — changes on every owner action; stale status would mislead borrowers
- Loan status — Out / Returned / Overdue must always reflect the live database value
- Member identity fields — never cached; always derived from the JWT at request time
### Manual response (who is paged, what they do)
 
1. **On-call engineer** receives a Severity 2 Teams alert — expected acknowledgement within 15 minutes.
2. Open Azure Portal → Azure Cache for Redis → Overview → check provisioning state and memory pressure.
3. If Redis is in a failed/degraded state → Resource menu → **Reboot** → select Primary node reboot.
4. If reboot does not recover within 5 minutes → check if the SKU is Basic tier; Basic has no replica. If so, scale up to Standard tier which supports auto-failover.
5. Monitor Azure SQL DTU in the portal — a cold cache pushes all reads to SQL; if DTU is above 80%, temporarily scale up SQL compute.
6. Post in `#bookswap-status`: _"Search is running slower than usual while we recover the cache. No data loss, results are accurate."_
7. Confirm Redis recovery by re-running the detection query and verifying `cachehits` is non-zero.
### Post-incident actions
 
- [ ] Upgrade Redis from Basic to Standard tier — Standard has a replica and supports automatic failover with no manual intervention
- [ ] Add a cache warm-up job that pre-loads the top 50 most-searched books into Redis on every App Service startup
- [ ] Add a custom Application Insights metric `cache_fallback_rate` so the SLO dashboard shows when the app is running cache-cold
- [ ] Verify that the 800 ms SLO held during this outage by querying SLI 1 for the outage window; if it did not hold, file a budget burn entry in the SLO record
- [ ] Document the Redis SKU requirement (Standard minimum) in the infrastructure checklist so it is not deployed as Basic in future environments
---
 
## Failure 3: Sunday tabloid spike — 10× sustained traffic
 
### What the user sees
 
- For the first 5–10 minutes while autoscale adds instances, `GET /books` may take 400–700 ms — still within SLO but noticeably slower
- Members submitting a new listing during peak may receive a `429 Too Many Requests` if they exceed the per-IP rate limit; the response includes a `Retry-After: 60` header
- Search (`GET /books`) stays fast because Redis absorbs the read traffic
- `POST /books` eventually succeeds — the idempotency key ensures a safe retry
- Email notifications for new listings are delayed by up to 2 hours while the Service Bus queue drains — this is expected and acceptable
### Detection
 
Request rate on `GET /books` crossing 5× the 28-day median RPS for 3 consecutive minutes triggers a Severity 2 alert.
 
```kusto
let baseline = toscalar(
  requests
  | where timestamp between (ago(28d) .. ago(1h))
  | where name == "GET /books"
  | summarize count() / (28.0 * 24 * 60)
);
requests
| where timestamp > ago(10m)
| where name == "GET /books"
| summarize rps = count() / 60.0 by bin(timestamp, 1m)
| where rps > 5 * baseline
```
 
Alert condition: 3 consecutive 1-minute bins above threshold → Severity 2 → Teams `#bookswap-ops`.
 
Secondary signals:
- App Service CPU > 70% for 5 minutes → autoscale should already be responding
- Azure SQL DTU > 80% for 5 minutes → risk of write throttling on `POST /books`
- Service Bus queue depth > 5 000 messages → email consumer is falling behind
### Mitigation in design (autoscale, queue depth, throttling)
 
**Azure Front Door WAF rate limiting:**
 
```
FOR every request to POST /books:
  COUNT requests from this client IP in the last 60 seconds
  IF count > 100:
    RETURN 429 Too Many Requests
    SET response header: Retry-After: 60
  ELSE:
    FORWARD request to App Service
 
FOR GET /books:
  COUNT requests from this client IP in the last 60 seconds
  IF count > 300:
    RETURN 429
    SET response header: Retry-After: 60
```
 
**App Service autoscale rules:**
 
```
CHECK every 5 minutes:
 
  Scale OUT rule:
    IF average CPU > 70% across all instances for 5 minutes:
      ADD 2 instances
      MINIMUM instances = 2
      MAXIMUM instances = 10
      COOLDOWN after scale-out = 5 minutes
 
  Scale IN rule:
    IF average CPU < 30% across all instances for 20 minutes:
      REMOVE 1 instance
      COOLDOWN after scale-in = 10 minutes
```
 
**Idempotency key for listing creation (prevents duplicates on 429 retry):**
 
```
WHEN POST /books is received:
  READ Idempotency-Key header (UUID v4, client-generated, required during spike)
 
  IF key exists in idempotency_keys table (TTL 24 hours):
    RETURN the original 201 response body
    DO NOT create a second book record
 
  IF key is absent:
    RETURN 400 Bad Request: "Idempotency-Key header required"
 
  IF key is new:
    BEGIN transaction
      INSERT book record
      INSERT idempotency_keys (key, response_body, expires_at = now + 24h)
    COMMIT
    RETURN 201 Created
```
 
**Service Bus queue decouples email sends from the request path:**
 
```
WHEN POST /books succeeds:
  SAVE book to Azure SQL
  PUBLISH BookListed message to Service Bus queue (fire and forget)
  RETURN 201 immediately — do not wait for email
 
Email consumer (separate background worker):
  POLL Service Bus queue continuously
  FOR each BookListed message:
    SEND email digest to building residents
    COMPLETE the message (removes it from queue)
    IF send fails → ABANDON message (Service Bus redelivers up to 5 times)
    IF 5 attempts all fail → message moves to dead-letter queue for manual review
 
During 10× spike:
  Queue depth grows but the API response time is unaffected
  Consumer drains the queue at its own pace after the spike subsides
```
 
**Redis absorbs the read spike:**
 
```
Cache TTL for GET /books list = 30 seconds
Cache TTL for GET /books/{bookId} = 60 seconds
 
AT 10× RPS most GET /books requests are served from Redis
Azure SQL only sees cache-miss traffic (first request per 30-second window)
This keeps SQL DTU well below the throttle threshold during the spike
```
 
### Manual response (who is paged, what they do)
 
1. **On-call engineer** receives a Severity 2 Teams alert — acknowledge within 15 minutes.
2. Open Azure Portal → App Service → Scale out → confirm autoscale is active and instance count is increasing.
3. If autoscale is not triggering within 10 minutes → manually set instance count to **8** via the portal Scale out blade.
4. Open Azure Portal → Azure SQL → Metrics → DTU percentage. If DTU > 80% → scale up SQL compute tier temporarily (e.g. from GP_Gen5_2 to GP_Gen5_4).
5. Open Azure Portal → Service Bus → `bookswap-listings` queue → check Active Message Count. If queue depth > 10 000 → increase email consumer concurrency from 1 to 4 workers via App Service environment variable `EMAIL_CONSUMER_CONCURRENCY=4` and restart the consumer.
6. Open Azure Front Door → WAF Logs → confirm `429` responses are being returned to clients exceeding the rate limit; confirm legitimate traffic is passing through.
7. Post status updates in `#bookswap-status` every 30 minutes while the spike is active: include current RPS, instance count, SQL DTU, and queue depth.
8. After the spike subsides (RPS back to baseline for 30 minutes) → confirm autoscale scales back to minimum 2 instances; if not, manually reduce to avoid unnecessary cost.
### Post-incident actions
 
- [ ] Run a load test at 10× median RPS using Azure Load Testing before the next expected Sunday spike — verify autoscale completes within 8 minutes and the 800 ms SLO holds throughout
- [ ] Switch the autoscale trigger from CPU percentage to **HTTP queue length** — CPU lags behind request queue growth by several minutes; queue length reacts in under 60 seconds
- [ ] Add an Azure SQL read replica and route all `GET /books` SQL fallback queries to the replica — this removes read pressure from the primary during cache-cold + spike scenarios
- [ ] Review Service Bus dead-letter queue after every spike — if messages are present, investigate the email consumer failure and fix before the next spike
- [ ] Add a pre-warm scheduled action in autoscale: every Sunday at 07:00 set minimum instances to 4 so scale-out starts from a higher floor and the first wave of users does not experience the warm-up lag
 