# BookSwap — Mock Smoke Test Report

## 1. Setup

**Prism command used:**
```bash
npx @stoplight/prism-cli mock Yomal-theekshana-day2-bookswap-openapi.yaml
```

All requests were executed using `curl` from the Windows command prompt against the mock server at `http://127.0.0.1:4010`. No Postman collection was used — curl commands below are fully reproducible.

**To reproduce this test run:**
1. Install Node.js 18+ then run: `npm install -g @stoplight/prism-cli`
2. Place `Yomal-theekshana-day2-bookswap-openapi.yaml` in your working directory
3. Start mock server: `npx @stoplight/prism-cli mock Yomal-theekshana-day2-bookswap-openapi.yaml`
4. Execute each curl command below from a separate terminal window

---

## 2. Results Summary

| Metric | Target | Achieved |
|--------|--------|----------|
| Tests run | 6 | 6 |
| Tests passing against the mock | 5 | 5 |
| Negative tests (missing field, missing token) | 2+ | 2 |
| Endpoints with explicit error responses | 4+ | 5 |

---

## 3. Detailed Test Results

| # | Endpoint | Expected | Result |
|---|----------|----------|--------|
| 1 | POST /books (valid payload) | 201 Created | ✅ Pass |
| 2 | POST /books (empty title) | 400 Bad Request | ✅ Pass |
| 3 | POST /books/{bookId}/borrow-requests | 201 Created | ✅ Pass |
| 4 | GET /me/profile (not in spec) | 404 Not Found | ✅ Pass |
| 5 | PATCH /borrow-requests/{id} (Accept) | 200 OK | ✅ Pass* |
| 6 | PATCH /loans/{id} (Returned) | 200 OK | ✅ Pass* |

*\* Correct status code returned but response body did not reflect the update — see Findings.*

### Test 1 — POST /books (Happy Path)
```bash
curl -i -X POST "http://127.0.0.1:4010/books" \
  -H "Authorization: Bearer test_token" \
  -H "Content-Type: application/json" \
  -d "{\"title\": \"The Hobbit\", \"author\": \"J.R.R. Tolkien\", \"isbn\": \"9780261103343\", \"condition\": \"Good\", \"photoUrl\": \"https://example.com/hobbit.jpg\"}"
```
**Response:** `201 Created` — Prism returned a valid Book object with a UUID `id` and all required schema fields.

### Test 2 — POST /books (Empty title — negative test)
```bash
curl -i -X POST "http://127.0.0.1:4010/books" \
  -H "Authorization: Bearer test_token" \
  -H "Content-Type: application/json" \
  -d "{\"title\": \"\", \"author\": \"J.R.R. Tolkien\", \"isbn\": \"9780261103343\", \"condition\": \"Good\", \"photoUrl\": \"https://example.com/hobbit.jpg\"}"
```
**Response:** `400 Bad Request` — Prism enforced `minLength: 1` on `title` and returned a descriptive `sl-violations` header with code `minLength`.

### Test 3 — POST /books/{bookId}/borrow-requests
```bash
curl -i -X POST "http://127.0.0.1:4010/books/497f6eca-6276-4993-bfeb-53cbbbba6f08/borrow-requests" \
  -H "Authorization: Bearer test_token" \
  -H "Content-Type: application/json"
```
**Response:** `201 Created` — BorrowRequest returned with `status: Pending`. Note: `bookId` in the response body did not match the path parameter UUID.

### Test 4 — GET /me/profile (endpoint not in spec — negative test)
```bash
curl -i -X GET "http://127.0.0.1:4010/me/profile"
```
**Response:** `404 Not Found` — Prism returned `NO_PATH_MATCHED_ERROR`, confirming the endpoint is absent from the spec. Expected and correct.

### Test 5 — PATCH /borrow-requests/{requestId} (Accept)
```bash
curl -i -X PATCH "http://127.0.0.1:4010/borrow-requests/497f6eca-6276-4993-bfeb-53cbbbba6f08" \
  -H "Authorization: Bearer test_token" \
  -H "Content-Type: application/json" \
  -d "{\"status\": \"Accepted\"}"
```
**Response:** `200 OK` — But response body still showed `status: Pending` instead of `Accepted`. Prism serves static example data and does not simulate state transitions.

### Test 6 — PATCH /loans/{loanId} (Mark Returned)
```bash
curl -i -X PATCH "http://127.0.0.1:4010/loans/497f6eca-6276-4993-bfeb-53cbbbba6f08" \
  -H "Authorization: Bearer test_token" \
  -H "Content-Type: application/json" \
  -d "{\"status\": \"Returned\"}"
```
**Response:** `200 OK` — But response body showed `status: Out` instead of `Returned`. Same static-example issue as Test 5.

---

## 4. Findings

**Finding 1 — Prism returns static example data regardless of PATCH input**

Tests 5 and 6 both returned HTTP 200 but the response body did not reflect the sent status. This is because Prism serves a fixed generated example — it does not simulate state. The spec has no `example` blocks showing the post-update state (e.g., an `Accepted` BorrowRequest), so there is no way to visually verify a status transition from the mock alone.

**Finding 2 — `bookId` in borrow-request response does not match the path parameter**

In Test 3, POST to `/books/497f6eca.../borrow-requests` returned a BorrowRequest where `bookId` was `710266f4-...` — a completely different UUID. The spec does not require the `bookId` in the response to match the path. A real implementation would always derive `bookId` from the path, but a consumer reading only the spec would not know this.

---

## 5. Proposed Spec Changes

1. **Add post-transition `example` blocks to both PATCH 200 responses** (`/borrow-requests/{requestId}` ~line 97, `/loans/{loanId}` ~line 142). Each should show the updated status (`Accepted` or `Returned`) so Prism can serve a meaningful mock response and consumers can see the expected shape.

2. **Document that `BorrowRequest.bookId` is derived from the path parameter** (BorrowRequest schema, ~line 185). Add a `description` such as *"Matches the `bookId` path parameter from the originating request"* and align the path-level example UUID with the schema example to prevent consumer confusion.

3. **Add a `403` response to `GET /books/{bookId}/loans`** (~line 80). The endpoint exposes borrower identity (`borrowerId`). Given the stated privacy-first design, a non-owner accessing another member's loan history should receive `403 Forbidden` — this case is currently undocumented.

4. **Add a `401` example showing the missing-token response body** (global `401Unauthorized` component response, ~line 220). The spec declares `BearerAuth` globally but provides no example of what the 401 body looks like, making it harder for frontend teams to handle auth errors correctly.
