# BookSwap — SLI/SLO Map

## 1. NFR Inventory

| # | NFR | User-visible behaviour |
|---|-----|------------------------|
| 1 | 99% of catalogue searches complete in under 800 ms over 28 days | Member searches for a book and gets results without a noticeable wait |
| 2 | Listing creation succeeds 99.9% of the time; retries must not create duplicates | Member lists a book and it appears; retrying a failed request never creates a ghost duplicate |

---

## 2. SLI / SLO Table

| # | Plain-language SLI | SLO target | Window | Error budget |
|---|--------------------|------------|--------|--------------|
| 1 | Proportion of `GET /books` requests that succeed and respond in under 800 ms | 99% | Rolling 28 days | 1% — roughly 403 slow requests/day at 100 req/min |
| 2 | Proportion of `POST /books` requests that return a 2xx response | 99.9% | Rolling 28 days | 0.1% — roughly 1.4 failed listings per 28 days at 50 listings/day |

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
