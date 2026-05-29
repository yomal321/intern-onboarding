# GreenChit — Architecture Trade-off Table

## Options Under Review

**Option A: App Service Monolith**
The entire Claims API runs as a single Node.js/Express application on Azure App Service P2v3. Notification Worker and Export Worker run as Azure Functions.

**Option B: Container Apps Split**
The Claims API is split into smaller services, each containerised and deployed independently on Azure Container Apps with KEDA-based scaling.

---

## Trade-off Scoring Table

Score: 1 = poor, 5 = excellent

| Quality Attribute | Option A: App Service Monolith | Option B: Container Apps Split | Why |
|---|---|---|---|
| **Time-to-first-deploy** | 5 | 2 | Option A deploys with a single `az webapp deploy` command and no container tooling. Option B requires Dockerfiles, Azure Container Registry, Container Apps Environment setup, and KEDA scaling rules — significant setup time on a 6-week deadline. |
| **Cost (low spend)** | 5 | 2 | Option A runs on a fixed P2v3 plan with predictable monthly cost. Option B uses consumption-based billing that can be unpredictable — multiple containers each with their own scaling rules can spike costs unexpectedly for a small internal tool. |
| **Operability for 10-person team** | 4 | 3 | Option A is well-documented and the team already uses App Service at BISTEC — low operational overhead. Option B requires the team to understand container orchestration, KEDA triggers, and Container Apps networking — a steeper learning curve for a small team with no container experience. |
| **Independent deploy** | 1 | 5 | Option A deploys the entire API as one unit — a small change to the export logic requires redeploying the whole application. Option B allows each service to be deployed independently, reducing blast radius and enabling parallel team workflows. |
| **Future scaling** | 2 | 5 | Option A scales vertically or by adding App Service instances — less granular and less cost-efficient at scale. Option B scales each component independently based on queue depth or HTTP load — much better for a system where notification and export workloads grow at different rates. |
| **Authn/authz consistency** | 4 | 3 | Option A enforces JWT validation in a single Auth Middleware that covers all routes in one place — easy to audit and test. Option B requires each service to implement or share the same auth middleware — consistency must be enforced across multiple codebases, increasing the risk of a service being accidentally left unprotected. |
| **Total** | **21** | **20** | Option A wins by 1 point overall but the score gap is not the deciding factor — see Decision Rationale below. |

---

## Decision Rationale

**We choose Option A: App Service Monolith.**

The two attributes that drove this decision are:

**1. Time-to-first-deploy (score gap: 5 vs 2)**
The 6-week delivery deadline is a hard constraint. Option B's container setup overhead — Dockerfiles, Container Registry, KEDA rules, Container Apps Environment — would consume 1–2 weeks of the delivery window before a single line of business logic is deployed. This risk is unacceptable given the timeline.

**2. Operability for a 10-person team (score gap: 4 vs 3)**
The team has no Container Apps production experience. Introducing container orchestration on a first deployment increases the risk of operational incidents that the team is not equipped to debug quickly. App Service is already used at BISTEC — the team knows how to deploy, monitor, and roll back.

**Accepted trade-off:**
Option A scores poorly on Independent Deploy (1 vs 5) and Future Scaling (2 vs 5). These limitations are consciously accepted — at 200 BISTEC staff, independent deployment and horizontal scaling are not needed today. This decision will be revisited if GreenChit scales beyond BISTEC or the team gains container experience.
