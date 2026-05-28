# ADR 0004: Use Microsoft Entra ID for Authentication

## 1. Status (current decision state)

Accepted

Date: 2026-05-28

## 2. Context (why this decision was needed)

The team needed to choose an authentication strategy for GreenChit.

Options compared:

- Microsoft Entra ID (SSO)
- Custom username and password authentication
- Third-party auth provider (Auth0, Okta)

Project facts:

- BISTEC already uses Microsoft Entra ID as the company identity provider
- all staff already have Entra ID accounts — no separate account provisioning needed
- GreenChit is an internal tool — only BISTEC employees should access it
- the system has three distinct roles: Staff, Line Manager, Finance, and Audit
- privacy requirement states only the claimant, line manager, finance, and audit role may view a given claim
- the team has no experience building and maintaining custom auth systems
- security vulnerabilities in custom auth are a known and costly risk

## 3. Decision (what the team chose)

GreenChit will use Microsoft Entra ID (BISTEC tenant) for all authentication.

Frontend setup:

- MSAL.js library handles login flow in the React SPA
- users sign in with their existing BISTEC Microsoft account
- no separate GreenChit account or password needed

Backend setup:

- Claims API uses express-jwt middleware to validate JWT bearer tokens on every route
- JWKS endpoint from Entra ID used to verify token signatures
- memberId and role claims extracted from the validated JWT
- no tokens stored in the database — stateless validation on every request

Role mapping:

- roles defined in Entra ID App Registration
- Staff, Line Manager, Finance, and Audit roles assigned in Entra ID
- API enforces role checks per route

## 4. Consequences (benefits and risks)

Benefits:

- zero additional account management — staff use existing BISTEC credentials
- MFA and conditional access policies inherited from BISTEC Entra ID setup for free
- no password storage, no password reset flow, no credential leak risk in GreenChit
- role assignment managed centrally in Entra ID by the BISTEC IT admin
- stateless JWT validation keeps the API fast and horizontally scalable

Risks:

- GreenChit is fully dependent on Entra ID availability — if Entra ID has an outage, no one can log in; this is accepted because Microsoft's SLA exceeds our 99.9% requirement
- role changes in Entra ID take effect only after the user's token expires and they re-authenticate — there is a window where a role change is not immediately reflected
- the team must understand OAuth 2.0 and OIDC flows to debug auth issues — this is not zero learning curve

## 5. Alternatives Considered (other options rejected)

Option 1: Custom username and password authentication

Rejected because:

- requires building password hashing, storage, reset flows, and brute force protection
- security vulnerabilities in custom auth are a well-known risk
- BISTEC already has an identity provider — duplicating it adds no value
- maintenance burden is ongoing and disproportionate for an internal tool

Option 2: Third-party auth provider (Auth0 or Okta)

Rejected because:

- adds external vendor cost and dependency for a system that already has Entra ID
- requires mapping BISTEC users to a second identity system
- no advantage over Entra ID when BISTEC is already a Microsoft tenant

## 6. Simple Summary (one-line recap)

GreenChit will use Microsoft Entra ID because BISTEC staff already have accounts, roles can be managed centrally, and we inherit enterprise-grade security without building any auth infrastructure.
