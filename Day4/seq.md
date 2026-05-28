# Sequence — Submit and Approve a Claim

## Happy path

```mermaid
sequenceDiagram
  participant U as Claimant (browser)
  participant FE as Web App
  participant API as Claims API
  participant DB as Azure SQL
  participant BLOB as Blob Storage
  participant SB as Service Bus
  participant NW as Notification Worker
  participant TEAMS as Teams webhook
  participant MGR as Line Manager (Teams)

  U->>FE: Fill claim form, tap "Submit"
  FE->>API: POST /claims (JWT, claim metadata)
  API->>API: Auth Middleware — validate JWT, extract memberId
  API->>API: Approval Engine — assert Draft→Submitted is valid
  API->>DB: INSERT claim (status=Submitted) + INSERT audit_log row
  DB-->>API: claim_id returned
  API->>BLOB: Generate signed PUT URLs for receipt files
  BLOB-->>API: Signed URLs (15-min expiry)
  API-->>FE: 201 Created { claimId, signedUploadUrls[] }
  FE->>BLOB: PUT receipt files directly (multipart, signed URLs)
  BLOB-->>FE: 200 OK for each file
  FE->>API: PATCH /claims/{claimId} { status: "receipt_attached" }
  API->>DB: UPDATE claim — mark receipts complete
  API->>SB: Publish claim.submitted message (async)
  API-->>FE: 200 OK — claim fully submitted
  FE-->>U: "Claim submitted — your manager has been notified"

  Note over SB,NW: Asynchronous delivery — decoupled from HTTP response
  SB-->>NW: Deliver claim.submitted message
  NW->>TEAMS: POST Adaptive Card webhook (approve / reject / info)
  TEAMS-->>MGR: Notification card with action buttons

  MGR->>API: POST /claims/{claimId}/approve (JWT)
  API->>API: Auth Middleware — validate JWT, assert caller is line manager
  API->>API: Approval Engine — assert Submitted→Approved is valid
  API->>DB: UPDATE status=Approved + INSERT audit_log row
  DB-->>API: Success
  API->>SB: Publish claim.approved message (async)
  API-->>MGR: 200 OK { claimId, status: "Approved" }

  SB-->>NW: Deliver claim.approved message
  NW->>TEAMS: POST Adaptive Card to claimant — "Your claim was approved"
` `` 

---

## Error path 1 — receipt upload fails after the claim was created

` ``mermaid
sequenceDiagram
  participant U as Claimant (browser)
  participant FE as Web App
  participant API as Claims API
  participant DB as Azure SQL
  participant BLOB as Blob Storage

  U->>FE: Fill claim form, tap "Submit"
  FE->>API: POST /claims (JWT, claim metadata)
  API->>DB: INSERT claim (status=Submitted) + INSERT audit_log row
  DB-->>API: claim_id returned
  API->>BLOB: Generate signed PUT URLs
  BLOB-->>API: Signed URLs
  API-->>FE: 201 Created { claimId, signedUploadUrls[] }

  FE->>BLOB: PUT receipt file (intermittent Wi-Fi)
  alt Receipt upload fails (network error or timeout)
    BLOB-->>FE: connection reset / 5xx
    FE->>API: PATCH /claims/{claimId} { status: "upload_failed" }
    API->>DB: UPDATE status=Draft + INSERT audit_log row (reason="upload_failed")
    DB-->>API: Success
    API-->>FE: 200 OK { claimId, status: "Draft", retryUrls: [...] }
    FE-->>U: Upload failed — tap to retry your claim is saved as Draft
  else Receipt upload succeeds
    BLOB-->>FE: 200 OK
    FE->>API: PATCH /claims/{claimId} { status: "receipt_attached" }
    API-->>FE: 200 OK
  end
` ``

---

## Error path 2 — manager attempts to approve a claim they do not own

` ``mermaid
sequenceDiagram
  participant WRONGMGR as Other Manager (Teams)
  participant API as Claims API
  participant DB as Azure SQL

  WRONGMGR->>API: POST /claims/{claimId}/approve (JWT)
  API->>API: Auth Middleware — validate JWT, extract managerId
  API->>DB: SELECT claim.lineManagerId WHERE id = claimId
  DB-->>API: lineManagerId = "manager-A"
  API->>API: Assert caller.id == claim.lineManagerId → FAIL
  API-->>WRONGMGR: 403 Forbidden { code: "NOT_CLAIM_MANAGER" }
  Note over API,DB: No state change written; no audit row; no notification published
` ``
```

---

