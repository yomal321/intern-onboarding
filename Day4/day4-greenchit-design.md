# GreenChit — Architecture Design Pack
 
---
 
## 1. System Context
 
GreenChit is an internal web application built for BISTEC Global to manage staff expense reimbursements end-to-end. Staff submit expense claims with receipt images, line managers receive a Microsoft Teams notification and approve or reject each claim, and the Finance team exports approved claims as a CSV file that is automatically picked up by the payroll system. Every state change in the system — who did what, when, and why — is recorded in a tamper-evident audit log retained for seven years per company finance policy. The system is hosted entirely on Microsoft Azure, uses Microsoft Entra ID for single sign-on so staff log in with their existing BISTEC Microsoft accounts, and is designed to work smoothly on both desktop browsers and mobile devices.
 
---
 
## 2. Containers — C4 Level 2
 
![GreenChit Container Diagram](dirg/greenchit-container-diagram.drawio.svg)
 
### Container Table
 
| Container | Technology | Responsibility |
|---|---|---|
| **Web App** | React SPA, Azure Static Web Apps | Responsive UI served to all roles — staff, manager, finance, and auditor — on browser and mobile |
| **Claims API** | Node.js, Express, Azure App Service P2v3 | Handles all business logic — claim lifecycle, JWT validation, signed URL generation, audit logging, and CSV export triggering |
| **Database** | Azure SQL — General Purpose | Persists all claim records, audit log rows, and claimant data. Audit table is append-only with INSERT-only permissions |
| **Blob Storage** | Azure Blob Storage | Stores receipt image files up to 10 MB each. Files are accessed via short-lived signed SAS URLs — clients upload directly without going through the API |
| **Queue** | Azure Service Bus Standard | Holds async messages for notifications and export jobs. Decouples the API response from downstream processing to meet the 1.5 s p95 SLA |
| **Notif. Worker** | Azure Function — consumption plan | Consumes claim state change messages from the Queue and posts Adaptive Cards to Microsoft Teams. Falls back to email if Teams webhook is unavailable |
| **Export Worker** | Azure Function — consumption plan | Consumes export job messages from the Queue and generates a CSV file dropped into a SharePoint folder watched by the payroll automation |
| **Identity** | Microsoft Entra ID — BISTEC tenant | Issues and validates JWT tokens for all users. Manages role assignments for Staff, Line Manager, Finance, and Audit roles |
 
### Key Relationships
 
| From | To | Label | Protocol |
|---|---|---|---|
| Staff / Manager | Web App | uses via | HTTPS |
| Web App | Claims API | sends requests via | REST / HTTPS |
| Web App | Blob Storage | uploads receipts directly via | direct PUT SAS / HTTPS |
| Claims API | Blob Storage | issues signed SAS URL via | HTTPS / Azure SDK |
| Claims API | Database | reads and writes claims via | SQL / TDS |
| Claims API | Queue | publishes state change event via | AMQP |
| Claims API | Identity | validates JWT bearer token via | HTTPS / JWKS |
| Queue | Notif. Worker | consumes notification job via | AMQP |
| Queue | Export Worker | consumes export job via | AMQP |
 
---
 
## 3. Components — C4 Level 3: Claims API
 
![GreenChit Component Diagram](dirg/component-dig.drawio.svg)
 
### Component Table
 
| Component | Technology | Responsibility |
|---|---|---|
| **Auth Middleware** | express-jwt, Entra JWKS | Validates the JWT bearer token on every incoming route. Extracts memberId and role claims. Rejects any request with an invalid or expired token before it reaches any controller |
| **Claims Controller** | Express Router /claims | Handles all CRUD operations for claims. Enforces ownership checks — a staff member can only read and update their own claims. Triggers the Approval Engine on every status change and calls the Audit Writer to record the transition |
| **Receipt Orchestrator** | Signed URL generator | Generates short-lived SAS PUT URLs for direct client-to-blob uploads. Validates file size limit (10 MB) and file type before issuing the URL. Does not handle the file itself |
| **Export Controller** | Express Router /export | Finance-only route. Accepts the export trigger and returns 202 Accepted immediately. Hands off to the Notif. Publisher to enqueue the export job asynchronously |
| **Approval Engine** | State machine — pure function | Enforces the claim state machine rules. Asserts that transitions are valid — Draft to Submitted, Submitted to Approved or Rejected. Rejects any invalid state jump before the database is touched |
| **Audit Writer** | Append-only DB client | Inserts a new row into the audit_log table for every state transition. Uses an INSERT-only SQL login — it can never update or delete existing audit rows. Records actor, timestamp, from-state, to-state, and rejection reason |
| **Notif. Publisher** | Azure Service Bus SDK | Publishes claim event messages to the Service Bus queue asynchronously after every state change. Does not wait for delivery — fire and forget to keep the HTTP response fast |
| **Health Controller** | Express Router | Exposes /health and /health/db endpoints. Used by Azure App Service to monitor liveness and database connectivity |
 
### Key Relationships
 
| From | To | Label | Protocol |
|---|---|---|---|
| Web App | Auth Middleware | sends all requests via | HTTPS |
| Auth Middleware | Claims Controller | passes validated request to | internal |
| Auth Middleware | Receipt Orchestrator | passes validated request to | internal |
| Auth Middleware | Export Controller | passes finance role request to | internal |
| Claims Controller | Approval Engine | validates state transition via | internal |
| Claims Controller | Audit Writer | records state change via | internal |
| Approval Engine | Azure SQL | reads and writes claim state via | SQL / TDS |
| Audit Writer | Azure SQL | inserts audit record via | SQL / TDS |
| Receipt Orchestrator | Blob Storage | issues signed SAS URL via | HTTPS / Azure SDK |
| Export Controller | Notif. Publisher | enqueues export job via | internal |
| Notif. Publisher | Service Bus | publishes event via | AMQP |
 
---
 
## 4. Reading Order
 
Follow this order to walk through the diagrams as a reviewer.
 
---
 
### Step 1 — Start with the Container Diagram
 
**Look at the four Person nodes first**
Identify the four actors — Staff, Manager, Finance, and Audit Role. Understand who uses the system and for what purpose before looking at any technical detail.
 
**Follow the main request path left to right**
```
Staff / Manager → Web App → Claims API → Database
```
This is the core synchronous path — a user submits or approves a claim, the API validates it, and the database persists it.
 
**Follow the receipt upload path**
```
Web App → Blob Storage (direct PUT via signed URL)
Claims API → Blob Storage (issues the signed URL)
```
Note that the file never passes through the API — the client uploads directly to Blob Storage. This is the key design decision that satisfies the intermittent connectivity and 10 MB file size requirements.
 
**Follow the async notification path**
```
Claims API → Queue → Notif. Worker → Teams
                   → Export Worker → SharePoint → Payroll
```
Note the dashed arrows — these are asynchronous. The API drops a message and returns immediately. The workers process in the background. This is what keeps the submission round-trip under 1.5 seconds.
 
**Look at Identity last**
```
Claims API → Identity (validates JWT / HTTPS JWKS)
```
Every request passes through this check but it is shown at the bottom to keep the main flow readable.
 
---
 
### Step 2 — Move to the Component Diagram
 
**Start at Auth Middleware — the front door**
Every request from the Web App enters here first. Nothing reaches any controller unless the token is valid.
 
**Follow the three paths Auth fans out to**
```
Auth Middleware → Claims Controller     (staff and manager actions)
Auth Middleware → Receipt Orchestrator  (receipt upload requests)
Auth Middleware → Export Controller     (finance export — role restricted)
```
 
**Follow the Claims path downward**
```
Claims Controller → Approval Engine  (validates state transition)
Claims Controller → Audit Writer     (records every state change)
Approval Engine  → Azure SQL         (persists new claim state)
Audit Writer     → Azure SQL         (inserts audit row)
```
This is the most important path — it shows how the state machine and audit log work together.
 
**Follow the Receipt path to the right**
```
Receipt Orchestrator → Blob Storage  (issues signed SAS URL)
```
Short path — Receipt Orchestrator only generates the URL, it never touches the file.
 
**Follow the Export path to the right**
```
Export Controller → Notif. Publisher → Service Bus
```
Export Controller returns 202 immediately. The actual CSV generation happens downstream in the Export Worker — not shown here because that is in the Container diagram.
 
**Health Controller stands alone at the bottom**
No arrows to other components — it only checks that the API and database are reachable. Ignore it for the architecture review.
 
---
 
### Notation Guide
 
| Arrow style | Meaning |
|---|---|
| Solid arrow with verb label and protocol | Synchronous call — caller waits for response |
| Dashed arrow with verb label and protocol | Asynchronous message — caller does not wait |
| Arrow pointing to self | Internal processing — no external call |
 
> **One rule to remember:** Every arrow in both diagrams has a verb-led label (sends, validates, publishes, inserts) and a protocol (HTTPS, SQL/TDS, AMQP, internal). If an arrow has no label or no protocol it is incomplete.
 
