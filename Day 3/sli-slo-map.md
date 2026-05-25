# BookSwap — SLI/SLO Map

## 1. NFR Inventory

| # | NFR (from Day 2 + Day 3) | User-visible behaviour |
|---|--------------------------|------------------------|
| 1 | Catalogue search: 99% of requests under 800 ms over a rolling 28 days, even at 10× normal RPS | Member searches for a book and gets results without a noticeable wait |
| 2 | Listing creation: 99.9% success rate; failed attempts must be retryable without creating duplicates | Member lists a book and it appears in the catalogue; retrying a failed request never creates a ghost duplicate |
| 3 | Authentication: every endpoint except `/health` requires a valid JWT; tokens expire within 1 hour | Member is never served another person's data; stale sessions are rejected automatically |
| 4 | Detection: a complete outage of the listings endpoint must page on-call within 3 minutes | Operations team can confirm system health in under 5 minutes; prolonged outages are not discovered by users first |
| 5 | Audit: every auth failure and every loan creation/return is logged with request ID and member ID | Disputes about loan history or unauthorised access can be investigated and resolved |
| 6 | System keeps accepting new listings during a Sunday 10× RPS spike sustained 4 hours | Members can list books on the busiest day without seeing errors or timeouts |
| 7 | Search results remain useful when cache is cold or unavailable | Member still gets correct (if slightly slower) results even during a Redis outage |
| 8 | A member must never see another member's loan history or address | Browsing, searching, or requesting loan data never leaks another resident's private records |

---

## 2. SLI Plain-Language Definitions

> **Hint:** Define an SLI in plain language first, then code it.

| # | Plain-language definition | KQL / metric expression |
|---|---------------------------|-------------------------|
| 1 | "Is the book search fast enough?" — out of every 100 search requests, how many came back in under 800 ms? | `requests \| where name == "GET /books" \| summarize sli = countif(duration <= 800) * 100.0 / count()` |
| 2 | "Does listing a book actually work?" — out of every 100 attempts to add a book, how many succeeded without a server error? | `requests \| where name == "POST /books" \| summarize sli = countif(resultCode !startswith "5") * 100.0 / count()` |
| 3a | "Does the API turn away strangers?" — out of every 100 requests arriving without a valid token, how many were correctly rejected with 401? | `requests \| where tobool(customDimensions.hasValidToken) == false \| summarize sli = countif(resultCode == "401") * 100.0 / count()` |
| 3b | "Does the API refuse stale passes?" — out of every 100 accepted tokens, how many were issued within the last 60 minutes? | `requests \| where isnotempty(customDimensions.jwt_age_seconds) \| summarize sli = countif(toint(customDimensions.jwt_age_seconds) <= 3600) * 100.0 / count()` |
| 4 | "Is the listings page reachable right now?" — out of every 100 synthetic pings, how many got a healthy response within 5 seconds? | `availabilityResults \| summarize sli = countif(success == 1) * 100.0 / count()` |
| 5 | "Did we write down what happened?" — out of every 100 auditable events, how many produced a log entry with both a request ID and a member ID? | `AppTraces \| where Properties.eventType in ("LoanCreated","LoanReturned","AuthFailure") \| summarize sli = countif(isnotempty(Properties.requestId) and isnotempty(Properties.memberId)) * 100.0 / count()` |
| 6 | "Does listing still work when everyone is online?" — same as SLI 2 but only during 10× traffic windows | Same KQL as SLI 2, filtered to hours where request count exceeds 10× the 28-day median hourly count |
| 7 | "Does search degrade gracefully when the cache is down?" — out of every 100 search requests while Redis was unhealthy, how many still returned a valid result? | `requests \| where name == "GET /books" and tobool(customDimensions.cacheAvailable) == false \| summarize sli = countif(resultCode startswith "2") * 100.0 / count()` |
| 8 | "Did any member see someone else's records?" — out of every 100 loan-history responses, how many leaked a foreign member ID? | `customEvents \| where name == "PrivacyViolation" \| count` — target is always 0; any non-zero result is a severity-1 incident |

---

## 3. SLI / SLO Table

| # | SLI definition | Measurement source | SLO target | Window | Error budget |
|---|----------------|--------------------|------------|--------|--------------|
| 1 | **Search latency** — proportion of `GET /books` requests where server-side response time (measured at App Service, excluding client network) is ≤ 800 ms | Azure Application Insights — `requests` table, filter `name = "GET /books"`, field `duration` | 99% of requests ≤ 800 ms | Rolling 28 days | 1% of requests may exceed 800 ms — at 100 req/min that is ~403 slow requests per 28 days |
| 2 | **Listing creation success rate** — proportion of `POST /books` requests that return HTTP 2xx (including idempotent retries that return the existing resource) | Application Insights — `requests` table, filter `name = "POST /books"`, count non-5xx / total | 99.9% success | Rolling 28 days | 0.1% failure — at 50 listings/day that is ~1.4 failed listings per 28 days |
| 3a | **Auth rejection correctness** — proportion of requests to protected endpoints carrying an expired or absent JWT that return HTTP 401 (not 200 or 500) | Application Insights — custom event `AuthFailure`, joined to `requests` by `operation_Id`; ratio of `401` to total requests-without-valid-token | 99.9% of unauthenticated requests return 401 | Rolling 7 days | 0.1% — any leaked 200 on a protected route is a severity-1 incident regardless of budget |
| 3b | **Token TTL enforcement** — proportion of JWTs accepted by the API that were issued more than 60 minutes before the request timestamp | Application Insights — custom dimension `jwt_age_seconds` on each authenticated request; count where `jwt_age_seconds > 3600` / total authenticated requests | 0% of accepted tokens are older than 60 minutes | Rolling 7 days | Zero tolerance — a single acceptance of an expired token triggers immediate investigation |
| 4 | **Listing endpoint availability** — proportion of 1-minute synthetic ping intervals to `GET /books?pageSize=1` that return HTTP 2xx within 5 s | Azure Monitor — URL ping availability test on `GET /books?pageSize=1`, 1-minute cadence from two Azure regions | 99.5% availability | Rolling 28 days | 0.5% — ~201 minutes of downtime per 28 days; alert fires when 3 consecutive pings fail (≤ 3 min to page) |
| 5 | **Audit log completeness** — proportion of loan-create, loan-return, and auth-failure events where a corresponding structured log entry (containing `requestId` and `memberId`) appears in Azure Monitor Logs within 2 minutes of the event | Azure Monitor Logs — query `AppTraces` for `Properties.eventType in ("LoanCreated","LoanReturned","AuthFailure")`, join to `requests` on `operation_Id`, check `requestId` and `memberId` fields are non-null | 99.9% of qualifying events produce a complete log entry | Rolling 7 days | 0.1% — roughly 1 missing log entry per 1 000 auditable events |
| 6 | **Listing creation error rate under spike** — proportion of `POST /books` requests that return 5xx during any 4-hour window where RPS is ≥ 10× the 28-day median RPS | Application Insights — same as SLI 2, windowed to spike periods identified by `requests` count > 10× p50 hourly rate | ≤ 0.1% 5xx during spike window | Per spike event (4-hour window) | Same absolute budget as SLI 2; spike window is evaluated separately to surface degradation that averages away |
| 7 | **Cache-fallback search correctness** — proportion of `GET /books` requests that return a valid (non-empty schema-conformant) response body regardless of Redis availability | Application Insights — custom dimension `cacheHit` on search responses; ratio of valid 2xx responses to total requests, filtered to periods where Redis dependency calls show `success = false` | 100% of search requests return a valid response during Redis unavailability | Rolling 28 days | Zero tolerance — a search returning 500 because Redis is down is a design defect, not a budget item |
| 8 | **Privacy isolation** — proportion of `GET /loans` or `GET /books/{id}/loans` responses where the returned `borrowerId` values include any member ID other than the authenticated caller's own ID (for borrower-scoped queries) | Application Insights — custom event `PrivacyViolation` emitted by the authorisation middleware on any mismatch; count of such events | 0 privacy violations | Rolling 28 days | Zero tolerance — any single violation is a severity-1 incident; budget concept does not apply |

---

## 4. Error Budget Policy

### When a budget is exhausted (or forecast to exhaust before window end)

**SLI 1 — Search latency budget ≥ 50% consumed with > 14 days remaining in window:**
The team freezes all non-critical feature work on the search path. The on-call engineer opens a P1 performance investigation within 1 business day. No new Redis eviction policy changes or index modifications are deployed until latency is back within budget.

**SLI 2 / SLI 6 — Listing success rate budget exhausted:**
All deployments to the listing creation path are halted immediately. A rollback to the last known-good release is performed if the failure rate is still rising 30 minutes after detection. New feature releases resume only after 48 hours of clean telemetry at or above SLO.

**SLI 4 — Availability budget ≥ 75% consumed:**
Change freeze on the App Service and Front Door configuration. All deployments require a second approver and must be deployed to a staging slot with a 15-minute bake time before slot swap.

**SLI 5 — Audit log completeness budget exhausted:**
The logging pipeline is treated as a production incident. No loan creation or return operations are accepted until logging is confirmed healthy — the affected endpoints return `503 Service Unavailable` with a `Retry-After` header rather than silently drop audit events.

### Who owns the decision

The **on-call engineer** declares budget exhaustion and initiates the freeze. Lifting the freeze requires sign-off from the **engineering lead**. For SLIs 3b and 8 (zero-tolerance), the **security lead** is paged immediately regardless of time of day.

---

## 5. Out of Budget Right Now

**SLI 7 (cache-fallback search correctness)** is the one we almost certainly cannot meet today — the OpenAPI spec and Day 2 architecture have no documented fallback path from Redis to direct Azure SQL query, so a Redis outage right now would cause `GET /books` to return 500s until the cache is manually flushed or the service restarted, violating the zero-tolerance target on day one.
