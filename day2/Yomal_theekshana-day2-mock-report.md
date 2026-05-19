# BookSwap — Mock Smoke Test Report

## Setup
- **Prism command used:** `npx @stoplight/prism-cli mock openapi/bookswap-openapi.yaml -p 4010`
- **Reproduction:** Tests were executed via standard `curl` commands against the local Prism server to ensure another intern can cleanly reproduce them without needing a specific Postman file export.

### Executed cURL Commands (Reproducible Test Suite)
1. **GET /books (Valid pagination):**
   `curl -i -X GET "http://127.0.0.1:4010/books?page=1&pageSize=20" -H "Authorization: Bearer mock-jwt-token"`
2. **POST /books (Valid payload):**
   `curl -i -X POST "http://127.0.0.1:4010/books" -H "Authorization: Bearer mock-jwt-token" -H "Content-Type: application/json" -d '{"title": "The Hobbit", "author": "J.R.R. Tolkien", "condition": "Good"}'`
3. **POST /books (Missing title - Negative Test):**
   `curl -i -X POST "http://127.0.0.1:4010/books" -H "Authorization: Bearer mock-jwt-token" -H "Content-Type: application/json" -d '{"author": "J.R.R. Tolkien"}'`
4. **POST /books/{bookId}/borrow-requests (Valid JWT):**
   `curl -i -X POST "http://127.0.0.1:4010/books/999/borrow-requests" -H "Authorization: Bearer mock-jwt-token" -H "Content-Type: application/json" -d '{"durationDays": 14}'`
5. **GET /books (Missing token - Negative Test):**
   `curl -i -X GET "http://127.0.0.1:4010/books"`

## Results Summary

| Metric | Target | Achieved |
|--------|--------|----------|
| Tests run | 5 | 5 |
| Tests passing against the mock | 5 | 5 |
| Endpoints with explicit error responses | 4+ | 5 |

### Detailed Test Outputs

| # | Endpoint | Method | Body / Params | Expected status | Actual Status | Pass/Fail |
|---|----------|--------|---------------|-----------------|---------------|-----------|
| 1 | `/books` | GET | `page=1&pageSize=20` | 200 | 200 | ✅ Pass |
| 2 | `/books` | POST | valid book payload | 201 | 201 | ✅ Pass |
| 3 | `/books` | POST | missing title | 400 or 422 | 422 | ✅ Pass |
| 4 | `/books/999/borrow-requests` | POST | borrower JWT | 201 | 201 | ✅ Pass |
| 5 | `/books` | GET | (no Authorization header) | 401 | 401 | ✅ Pass |


## Spec changes you would make

- **1. Format Enforcement for URLs:** Under `components/schemas/BookListing/properties/photoUrl`, add `format: uri`. This tells Prism (and any code generators) to output a valid URL instead of a random 500-character alphanumeric string.
- **2. Standardized Error Responses:** Define a reusable error schema under `components/schemas/ErrorResponse` (containing `code`, `message`, and `details`). Then, apply this schema using `$ref` to the `400`, `401`, and `422` response definitions globally so that negative tests return a helpful JSON body instead of just an empty HTTP status code.