# BookSwap — SLI/SLO Map

## 1. NFR Inventory

| # | NFR | User-visible behaviour |
|---|-----|------------------------|
| 1 | 99% of catalogue searches complete in under 800 ms over 28 days | Member searches for a book and gets results without a noticeable wait |
| 2 | Listing creation succeeds 99.9% of the time; retries must not create duplicates | Member lists a book and it appears; retrying a failed request never creates a ghost duplicate |

---

## 2. SLI / SLO Table

| # | SLI definition | Measurement source | SLO target | Window | Error budget |
|---|----------------|--------------------|------------|--------|--------------|
| 1 | **Search latency** — proportion of `GET /books` requests where server-side response time is ≤ 800 ms | Azure Application Insights — `requests` table, filter `name = "GET /books"`, field `duration` | 99% of requests ≤ 800 ms | Rolling 28 days | 1% of requests may exceed 800 ms — at 100 req/min that is ~403 slow requests per 28 days |
| 2 | **Listing creation success rate** — proportion of `POST /books` requests that return HTTP 2xx (including idempotent retries that return the existing resource) | Application Insights — `requests` table, filter `name = "POST /books"`, count non-5xx / total | 99.9% success | Rolling 28 days | 0.1% failure — at 50 listings/day that is ~1.4 failed listings per 28 days |
| 3a | **Auth rejection correctness** — proportion of requests to protected endpoints carrying an expired or absent JWT that return HTTP 401 (not 200 or 500) | Application Insights — custom event `AuthFailure`, joined to `requests` by `operation_Id`; ratio of `401` to total requests-without-valid-token | 99.9% of unauthenticated requests return 401 | Rolling 7 days | 0.1% — any leaked 200 on a protected route is a severity-1 incident regardless of budget |
| 3b | **Token TTL enforcement** — proportion of JWTs accepted by the API that were issued more than 60 minutes before the request timestamp | Application Insights — custom dimension `jwt_age_seconds` on each authenticated request; count where `jwt_age_seconds > 3600` / total authenticated requests | 0% of accepted tokens are older than 60 minutes | Rolling 7 days | Zero tolerance — a single acceptance of an expired token triggers immediate investigation |
| 4 | **Listing endpoint availability** — proportion of 1-minute synthetic ping intervals to `GET /books?pageSize=1` that return HTTP 2xx within 5 s | Azure Monitor — URL ping availability test, 1-minute cadence from two Azure regions | 99.5% availability | Rolling 28 days | 0.5% — ~201 minutes of downtime per 28 days; alert fires when 3 consecutive pings fail (≤ 3 min to page) |
| 5 | **Audit log completeness** — proportion of loan-create, loan-return, and auth-failure events where a corresponding structured log entry (containing `requestId` and `memberId`) appears in Azure Monitor Logs within 2 minutes | Azure Monitor Logs — query `AppTraces` for `eventType in ("LoanCreated","LoanReturned","AuthFailure")`, check `requestId` and `memberId` fields are non-null | 99.9% of qualifying events produce a complete log entry | Rolling 7 days | 0.1% — roughly 1 missing log entry per 1 000 auditable events |
| 6 | **Listing creation error rate under spike** — proportion of `POST /books` requests that return 5xx during any 4-hour window where RPS is ≥ 10× the 28-day median RPS | Application Insights — same as SLI 2, windowed to spike periods where `requests` count > 10× p50 hourly rate | ≤ 0.1% 5xx during spike window | Per spike event (4-hour window) | Same absolute budget as SLI 2; spike window evaluated separately to surface degradation that averages away |
| 7 | **Cache-fallback search correctness** — proportion of `GET /books` requests that return a valid 2xx response during periods where Redis dependency calls show `success = false` | Application Insights — custom dimension `cacheHit` on search responses; ratio of valid 2xx to total requests filtered to Redis-down periods | 100% of search requests return a valid response during Redis unavailability | Rolling 28 days | Zero tolerance — a 500 because Redis is down is a design defect, not a budget item |
| 8 | **Privacy isolation** — count of `GET /loans` or `GET /books/{id}/loans` responses where returned `borrowerId` values include any member ID other than the authenticated caller's | Application Insights — custom event `PrivacyViolation` emitted by authorisation middleware on any mismatch | 0 privacy violations | Rolling 28 days | Zero tolerance — any single violation is a severity-1 incident |

### Application Insights query for SLI 1

```kusto
requests
| where timestamp > ago(28d)
| where name == "GET /books"
| summarize
    good  = countif(success == true and duration < 800),
    total = count()
| extend sli = 100.0 * good / total
```

### Application Insights query for SLI 2

```kusto
requests
| where timestamp > ago(28d)
| where name == "POST /books"
| summarize
    good  = countif(toint(resultCode) < 500),
    total = count()
| extend sli = 100.0 * good / total
```

### Application Insights query for SLI 1

```kusto
requests
| where timestamp > ago(28d)
| where name == "GET /books"
| summarize
    good  = countif(success == true and duration < 800),
    total = count()
| extend sli = 100.0 * good / total
```

### Application Insights query for SLI 2

```kusto
requests
| where timestamp > ago(28d)
| where name == "POST /books"
| summarize
    good  = countif(toint(resultCode) < 500),
    total = count()
| extend sli = 100.0 * good / total
```

---

## 3. Error Budget Policy

- **SLI 1 budget ≥ 50% consumed with > 14 days left** — freeze all changes to the search path; on-call opens a P1 performance investigation within 1 business day.
- **SLI 2 budget exhausted** — halt all deployments to the listing creation path immediately; rollback if failure rate is still rising after 30 minutes.
- **Who decides** — on-call engineer declares the freeze; engineering lead signs off before it is lifted.

---

## 4. Out of Budget Right Now

**SLI 1 (search latency)** — there is no Redis fallback path documented in the current spec, so a cold or unavailable cache would push p99 latency well past 800 ms on day one.
