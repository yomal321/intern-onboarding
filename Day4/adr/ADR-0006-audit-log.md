# ADR 0006: Implement Audit Log as an Append-Only SQL Table

## 1. Status (current decision state)

Accepted

Date: 2026-05-28

## 2. Context (why this decision was needed)

The team needed to decide how to implement the GreenChit audit log.

Options compared:

- Append-only table in Azure SQL with INSERT-only login
- Separate audit database or storage account
- Application-level soft deletes in the main claims table
- Azure Monitor / Log Analytics

Project facts:

- audit log must be tamper-evident per company finance policy
- audit records must be retained for 7 years
- every state transition must record who, when, from which state, to which state, and why if rejected
- only the audit role may view the full audit log
- the audit log must be queryable for compliance investigations
- the system already uses Azure SQL — adding a second storage system increases operational complexity

## 3. Decision (what the team chose)

The audit log will be implemented as an append-only table inside the existing Azure SQL database.

Table structure:

- audit_log_id — primary key
- claim_id — foreign key to claims table
- actor_id — Entra ID user who triggered the transition
- from_status — previous claim state
- to_status — new claim state
- reason — nullable, required when status is Rejected
- created_at — UTC timestamp, set by the database not the application

Access control:

- a dedicated SQL login with INSERT-only permissions is used by the Claims API for all audit writes
- the application login cannot UPDATE or DELETE any row in audit_log
- only the audit role SQL login can SELECT from audit_log
- no application code path exists to modify an existing audit row

Retention:

- Azure SQL long-term backup retention set to 7 years
- SQL Auditing enabled and streamed to Azure Blob Storage for an additional immutable layer

## 4. Consequences (benefits and risks)

Benefits:

- tamper-evidence is enforced at the database permission level — not just application convention
- audit log lives in the same database as claims — JOIN queries for compliance investigations are simple SQL
- no additional infrastructure needed — reuses the existing Azure SQL instance
- created_at set by the database server eliminates any risk of application clock manipulation
- INSERT-only login means even a compromised application cannot delete or alter audit history

Risks:

- the INSERT-only login is a second database credential to manage — if rotated incorrectly it breaks audit writes without breaking the main application; this is an operational risk the team must document and test
- audit log rows will accumulate for 7 years in the same database — table size must be monitored and archiving considered after year 3 or 4
- if Azure SQL itself is compromised at the infrastructure level, the audit log could theoretically be altered; the Azure Blob Storage SQL Audit stream provides a secondary record for this scenario but does not fully eliminate the risk

## 5. Alternatives Considered (other options rejected)

Option 1: Separate audit database or storage account

Rejected because:

- adds operational complexity with a second connection string, backup policy, and access control set
- cross-database queries for compliance investigations become harder
- the volume of audit data does not justify a separate system at BISTEC scale

Option 2: Application-level soft deletes in the main claims table

Rejected because:

- soft deletes in the same table as mutable claim data do not provide tamper-evidence
- nothing prevents the application from updating a soft-deleted row
- does not satisfy the finance policy requirement for a tamper-evident audit trail

Option 3: Azure Monitor / Log Analytics

Rejected because:

- Log Analytics is optimised for operational telemetry, not structured financial audit records
- querying claim state transitions via KQL is less natural than SQL for the Finance and Audit teams
- retention and export policies are less aligned with a 7-year finance compliance requirement

## 6. Simple Summary (one-line recap)

GreenChit will store the audit log as an append-only SQL table protected by an INSERT-only database login because it provides tamper-evidence at the database level, keeps audit data queryable alongside claims, and adds no new infrastructure.
