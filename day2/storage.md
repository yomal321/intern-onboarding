# BookSwap — Storage and Cache Decisions

## 1. Data inventory

| Data type           | Example record                                                                                                                                                                | Volume estimate (1y)                  | Read/write ratio |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | ---------------- |
| Book listing        | `{ id: "uuid-v4", title: "The Hobbit", author: "J.R.R. Tolkien", isbn: "9780261103343", condition: "Good", photoUrl: "https://...", isAvailable: true, ownerId: "user-123" }` | ~50,000 across all buildings          | read-heavy       |
| Member profile      | `{ memberId: "auth0\|65c...", fullName: "Kamal Perera", registeredAt: "2026-01-15T08:00:00Z" }`                                                                               | ~10,000 active members                | read-heavy       |
| Borrow request      | `{ requestId: "uuid-v4", bookId: "uuid-v4", requesterId: "user-789", status: "Pending", createdAt: "2026-05-18T14:30:00Z" }`                                                  | ~150,000 requests submitted           | balanced         |
| Loan tracker        | `{ loanId: "uuid-v4", bookId: "uuid-v4", borrowerId: "user-789", status: "Out", borrowedAt: "2026-05-18T14:32:00Z", dueDate: "2026-06-17T14:32:00Z", returnedAt: null }`      | ~120,000 active transitions           | balanced         |
| In-app notification | `{ notificationId: "uuid-v4", targetUserId: "user-123", message: "New borrow request", isRead: false, createdAt: "2026-05-18T14:30:05Z" }`                                    | ~500,000 alert entries                | write-heavy      |
| Book photo          | Raw JPEG/PNG image binary stream up to 5 MB max per upload limit.                                                                                                             | ~50,000 unique images (~250 GB total) | read-heavy       |

## 2. Storage selection

| Data type           | Chosen store       | Why this store                                                                                                                                                                                                     | Why not the alternatives                                                                                                                                                                                                                                                                                                                                                           |
| ------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Book listing        | Azure SQL          | **Source of Truth.** Relational setup with foreign keys (FK) to member profile table guarantees strict data models. Enforces instant transactional consistency, vital for booking stock status.                    | **Document DB (Cosmos DB):** Unnecessary operational complexity. Cross-document lookup joins linking user variables to books would require separate round-trip calls or complex map-reduce streams.<br><br>**Redis Cache:** Intended purely for fast memory caching. Any sudden container restart or patch operation completely wipes out the system's unpersisted inventory data. |
| Member profile      | Azure SQL          | **Source of Truth.** Enforces structured integrity rules for critical identity properties. Cleanly maps relational associations to books, borrow requests, and historical loan profiles using indexed constraints. | **Document DB (Cosmos DB):** Profile parameters match highly uniform structures. A document database is better suited for unstructured or wildly dynamic schemas, which do not exist here.                                                                                                                                                                                         |
| Borrow request      | Azure SQL          | **Source of Truth.** Highly relational transaction linkages that connect a borrowing neighbor directly to a targeted book instance, requiring precise constraint checking.                                         | **Document DB (Cosmos DB):** Lacks native multi-document immediate transaction constraints out of the box, introducing a severe race-condition risk of concurrent overlapping borrow attempts.                                                                                                                                                                                     |
| Loan tracker        | Azure SQL          | **Source of Truth.** Strict tabular schema maps cascade rules over historic records. Relational indices allow the application to process complex join configurations effortlessly.                                 | **Document DB (Cosmos DB):** Aggregating and tracking systemic multi-tenant history requires heavy table scans across isolated data streams rather than quick, precise relational index scans.                                                                                                                                                                                     |
| In-app notification | Azure Cosmos DB    | **Source of Truth.** Tailored for lightning-fast horizontal document insertion. Handles heavy alert generation tasks smoothly without risking thread resource exhaustion.                                          | **Azure SQL:** Rapid read/write notification loops (such as real-time unread tally updates) create heavy table lock contentions, stalling core business tables like checking out or returning books.                                                                                                                                                                               |
| Book photo          | Azure Blob Storage | **Source of Truth.** Standard storage engine designed specifically for cost-effective handling of massive, unstructured binary data chunks at scale.                                                               | **Azure SQL:** Appending up to 5 MB binary data blobs directly into structured columns splits data pages apart, causing severe index fragmentation, slow lookups, and massive system backup files.                                                                                                                                                                                 |

## 3. Cache plan

- **What is hot enough to cache?**
  To consistently meet the required **<300 ms p95 search performance target**, the application caches the global active book list index grids (the first 3 discovery pages of combined keyword/condition search configurations) and individual book detail profile blocks looked up frequently by identity strings.
  - **When NOT to cache (Critical Exception Case):** Loan Status Fields and Book Availability Flags (`isAvailable`) must **never** be cached. Caching availability states introduces an information lag window. If a user views a cached, stale flag, they can submit a borrow request for a book that is already checked out, causing database transaction failures and a broken user experience. Real-time availability must always query the relational source of truth directly.

- **Cache-aside pattern in pseudocode**
  ```javascript
  // Operational logic implementing standard Cache-Aside strategy
  async function getBookById(bookId) {
    const cacheKey = `book:details:${bookId}`;

    // Step 1: Query the fast memory caching tier (Azure Cache for Redis)
    let cachedData = await redis.get(cacheKey);
    if (cachedData !== null) {
      return JSON.parse(cachedData); // CACHE HIT: Returns instantly, bypassing primary database
    }

    // Step 2: CACHE MISS: Execute strict lookup across the primary relational database
    let databaseData = await azureSQL.query(
      "SELECT * FROM books WHERE id = ?",
      [bookId],
    );
    if (databaseData === null) {
      throw new Error("Target book record does not exist in inventory.");
    }

    // Step 3: Populate cache layer with an assigned Time-To-Live (TTL) configuration window
    await redis.setex(cacheKey, 3600, JSON.stringify(databaseData)); // Securely stored for 1 hour

    return databaseData;
  }
  ```