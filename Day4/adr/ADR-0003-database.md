# ADR 0003: Use Azure SQL as the Primary Database

## 1. Status (current decision state)

Accepted

Date: 2026-05-28

## 2. Context (why this decision was needed)

The team needed to choose a database for GreenChit claim records, audit logs, and user data.

Options compared:

- Azure SQL
- Azure Cosmos DB

Project facts:

- claims have a fixed, well-defined structure — claimant, manager, category, amount, status, receipts
- audit log must be tamper-evident and retained for 7 years per finance policy
- Finance team will run ad-hoc queries and CSV exports against approved claims
- strict state machine — Draft, Submitted, Approved, Rejected, Paid — requires referential integrity
- team has strong SQL skills and no Cosmos DB production experience
- around 200 BISTEC staff — no global distribution needed

## 3. Decision (what the team chose)

The primary database will be Azure SQL — General Purpose tier.

Schema areas:

- claims — claim records and line items
- audit_log — append-only, INSERT permissions only for the app user
- claimants — synced from Microsoft Entra ID

Data protection setup:

- audit_log protected by a dedicated SQL login with INSERT-only permissions
- row-level security enforces claim privacy — only claimant, line manager, finance, and audit roles can read a claim
- long-term backup retention set to 7 years to satisfy finance policy
- SQL Auditing enabled to Azure Storage for tamper-evidence

## 4. Consequences (benefits and risks)

Benefits:

- Finance team can write SQL directly for ad-hoc exports — no translation layer needed
- ACID transactions keep claim INSERT and audit_log INSERT atomic — no orphaned records
- team SQL skills mean faster development and fewer data access bugs
- row-level security and INSERT-only audit login are native SQL Server features

Risks:

- Azure SQL does not scale horizontally — if GreenChit grew to thousands of concurrent users across regions, re-architecture would be needed; accepted because BISTEC headcount makes this irrelevant today
- schema migrations require coordination and deployment windows — Cosmos DB's schema-less model would have made evolving the data structure easier
- the INSERT-only audit login adds a small amount of database administration overhead

## 5. Alternatives Considered (other options rejected)

Option 1: Azure Cosmos DB

Rejected because:

- data is relational with strict referential integrity requirements
- enforcing claim ownership and state transitions in application code instead of the database increases bug risk
- team has no Cosmos DB production experience — adds delivery risk
- global distribution and schema flexibility are not needed for this use case

Option 2: Azure Database for PostgreSQL

Rejected because:

- Azure SQL integrates more tightly with the Azure stack — Entra ID auth, Azure Monitor, long-term backup retention
- team already has Azure SQL experience
- would reconsider if cost becomes a concern at scale

## 6. Simple Summary (one-line recap)

GreenChit will use Azure SQL General Purpose because the data is relational, the team knows SQL, and the audit log tamper-evidence requirements are best met with native SQL Server features.
