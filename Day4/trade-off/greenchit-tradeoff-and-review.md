# GreenChit — Trade-offs and Design Review

## Setup

- Two architectural options under review
- Quality attributes weighted by team and business constraints
- Scoring: 1 = poor, 5 = excellent

---

## Options Under Review

**Option A: App Service Monolith**
The entire Claims API runs as a single Node.js/Express application on Azure App Service P2v3. Notification Worker and Export Worker run as Azure Functions.

**Option B: Container Apps Split**
The Claims API is split into smaller services, each containerised and deployed independently on Azure Container Apps with KEDA-based scaling.

---

## Trade-off Scoring Table

| Quality Attribute | Option A: App Service Monolith | Option B: Container Apps Split | Why |
|---|---|---|---|
| **Time-to-first-deploy** | 5 | 2 | Option A deploys with a single `az webapp deploy` command and no container tooling. Option B requires Dockerfiles, Azure Container Registry, Container Apps Environment setup, and KEDA scaling rules — significant setup time on a 6-week deadline. |
| **Cost (low spend)** | 5 | 2 | Option A runs on a fixed P2v3 plan with predictable monthly cost. Option B uses consumption-based billing — multiple containers each with their own scaling rules can spike costs unexpectedly for a small internal tool. |
| **Operability for 10-person team** | 4 | 3 | Option A is well-documented and the team already uses App Service at BISTEC — low operational overhead. Option B requires understanding container orchestration, KEDA triggers, and Container Apps networking — a steeper learning curve for a team with no container experience. |
| **Independent deploy** | 1 | 5 | Option A deploys the entire API as one unit — a small change to export logic requires redeploying the whole application. Option B allows each service to be deployed independently, reducing blast radius and enabling parallel team workflows. |
| **Future scaling** | 2 | 5 | Option A scales vertically or by adding App Service instances — less granular at scale. Option B scales each component independently based on queue depth or HTTP load — better for workloads that grow at different rates. |
| **Authn/authz consistency** | 4 | 3 | Option A enforces JWT validation in a single Auth Middleware covering all routes — easy to audit. Option B requires each service to implement the same auth middleware — consistency must be enforced across multiple codebases, increasing the risk of an unprotected service. |
| **Total** | **21** | **20** | Option A wins by 1 point overall. Score gap is not the deciding factor — see Decision Rationale. |

---

## Results Summary

| Metric | Target | Achieved |
|---|---|---|
| Quality attributes scored | 6 | 6 |
| Cells with a written justification | 12 | 12 |
| Decision-affecting attributes identified | 2–3 | 2 |

---

## Decision and Rationale

**We choose Option A: App Service Monolith.**

The two attributes that drove this decision are:

**1. Time-to-first-deploy (score gap: 5 vs 2)**
The 6-week delivery deadline is a hard constraint. Option B's container setup overhead would consume 1–2 weeks before a single line of business logic is deployed. This risk is unacceptable.

**2. Operability for a 10-person team (score gap: 4 vs 3)**
The team has no Container Apps production experience. App Service is already used at BISTEC — the team knows how to deploy, monitor, and roll back without specialist knowledge.

**Accepted trade-off:**
Option A scores poorly on Independent Deploy (1 vs 5) and Future Scaling (2 vs 5). These limitations are consciously accepted — at 200 BISTEC staff, independent deployment and horizontal scaling are not needed today. This decision will be revisited if GreenChit scales beyond BISTEC or the team gains container experience.

---

## Design Review Feedback — Received from Another Pair

### Strengths

1. **C4 diagrams are clear and consistent** — Container and Component diagrams use matching notation, arrow labels have protocols, and the boundary labels match between levels. Easy to follow as a reviewer.

2. **Audit log design is well thought out** — The INSERT-only SQL login is a strong tamper-evidence approach. Separating the audit credential from the main app credential shows real security awareness.

3. **Async decoupling via Service Bus is well justified** — The decision to decouple the notification from the HTTP response is clearly explained and directly tied to the 1.5 s p95 SLA requirement.

### Weaknesses and Risks

1. **The sequence diagram does not show the Finance export flow** — The happy path covers submit and approve but there is no sequence diagram for the Finance team triggering a CSV export. This is a required flow per the functional requirements and its absence is a gap.

2. **ADR-0002 does not define a rollback strategy** — The ADR says the team will use staging and production slots for zero-downtime deployment but does not describe what happens if a bad deploy reaches production. The swap-back procedure should be documented.

3. **Receipt upload retry is mentioned but not time-bounded** — The error path shows the claim stays in Draft with retry URLs but does not specify how long the retry URLs are valid or when a Draft claim expires. A claim stuck in Draft indefinitely could cause confusion for the Finance export.

### Actionable Improvements

1. **sequence-submit-approve.md** — Add a fourth diagram: `Error path 3 — Finance export flow` showing the Finance team triggering the CSV export, the Export Worker generating the file, and the SharePoint drop. This covers the missing functional requirement.

2. **ADR-0002-hosting-platform.md, Section 3 Decision** — Add a bullet point: `Rollback procedure: if a production slot swap causes issues, run az webapp deployment slot swap --slot staging to revert within 60 seconds`. This makes the deployment strategy complete and actionable.

---

## Design Review Feedback — Given to Another Pair

### Strengths

1. **Trade-off table covers all six required quality attributes** — Every cell has a score and a written justification. The scoring is honest and the reasoning is tied to real project constraints, not generic statements.

2. **ADR voice is consistently decisive** — Every ADR uses active voice: "we choose", "we adopt", "we will use". There is no hedging language like "we might" or "we plan to consider". This makes the decisions clear and auditable.

3. **Sequence diagrams correctly distinguish sync and async** — Solid arrows are used for synchronous calls and open async arrows are used for Service Bus publishes. The Note over SB,NW label correctly explains the decoupling. This matches the evaluation criteria exactly.

### Weaknesses and Risks

1. **Component diagram has an incorrect arrow** — The arrow from Receipt Orchestrator to Audit Writer is architecturally wrong. The Receipt Orchestrator only generates signed URLs — it does not trigger any state change and therefore should not write to the Audit Writer. This arrow should go from Claims Controller to Audit Writer.

2. **ADR-0003 does not address the 7-year retention cost** — The ADR mentions enabling long-term backup retention but does not estimate the storage cost over 7 years. For a finance compliance requirement this is a real operational risk that should be quantified or at least acknowledged.

3. **Container diagram is missing the Audit Role person** — The functional requirements define four actors: Staff, Line Manager, Finance, and Audit Role. The container diagram only shows Staff and Manager as a combined Person node. The Audit Role is a distinct actor with different access rights and should appear separately.

### Actionable Improvements

1. **greenchit-component-diagram.drawio** — Remove the arrow between Receipt Orchestrator and Audit Writer. Add a new arrow from Claims Controller to Audit Writer with the label `records state change / internal`. This corrects the architecture and aligns the component diagram with the sequence diagram.

2. **ADR-0003-database.md, Section 4 Consequences — Risks** — Add a bullet point: `Azure SQL long-term backup retention at 7 years for a growing audit_log table should be reviewed annually — estimated storage cost should be baselined in year 1 and monitored`. This addresses the missing cost risk for the finance compliance requirement.
