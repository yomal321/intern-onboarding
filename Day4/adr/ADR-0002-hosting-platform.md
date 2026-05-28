# ADR 0002: Host the Claims API on Azure App Service

## 1. Status (current decision state)

Accepted

Date: 2026-05-28

## 2. Context (why this decision was needed)

The team needed to choose a hosting platform for the GreenChit Claims API.

Options compared:

- Azure App Service
- Azure Container Apps
- Azure Kubernetes Service

Project facts:

- 6-week delivery timeline
- small team with no Container Apps experience
- around 200 BISTEC staff — low expected traffic
- 99.9% availability required during business hours
- 1.5 s p95 submission round-trip requirement
- App Service already used inside BISTEC

## 3. Decision (what the team chose)

The Claims API will run on Azure App Service P2v3.

Used for:

- Claims API — Node.js / Express backend

Background workers will run on Azure Functions consumption plan:

- Notification Worker
- Export Worker

Deployment setup:

- staging slot and production slot for zero-downtime releases
- Always On enabled to eliminate cold starts
- GitHub Actions for CI/CD pipeline

## 4. Consequences (benefits and risks)

Benefits:

- fast first deployment with no Docker setup needed
- team can focus on business features not infrastructure
- staging and production slots reduce release risk
- built-in App Insights for monitoring
- P2v3 Always On satisfies the 1.5 s SLA requirement

Risks:

- migrating to containers later will require pipeline refactoring — this cost is accepted now
- scaling is less flexible than Container Apps for unpredictable traffic
- team does not gain container experience during this project
- App Service is less portable if BISTEC moves off Azure

## 5. Alternatives Considered (other options rejected)

Option 1: Azure Container Apps

Rejected for v1 because:

- requires Dockerfiles, Container Registry, and KEDA scaling rules
- team has no prior Container Apps experience
- setup overhead would consume delivery time on a 6-week timeline

Option 2: Azure Kubernetes Service

Rejected because:

- too complex and too much operational overhead
- unnecessary for a small internal tool with bounded traffic
- team would spend more time managing infrastructure than building features

## 6. Simple Summary (one-line recap)

GreenChit will host the Claims API on Azure App Service P2v3 because it is faster, simpler, and safer for the first release given the team's experience and timeline.
