# User Career Profile



## Background

- **Experience:** 11+ years as a Software Engineer

- **Primary stack:** .NET (backend), currently on legacy .NET Framework (company constraint)

- **Database:** Some query optimization and schema design, wants to go deeper on data modeling

- **Cloud:** Azure — has built Azure Functions and done deployments, wants to go deeper

- **Frontend:** Likes JS, wants to learn (no current experience)

- **AI:** Uses AI tools (Claude, Copilot) daily. Has NOT built anything with AI APIs yet.



## Current Role Situation

- Purely execution-focused (no design ownership, no architecture decisions)

- Mostly bug fixing on legacy .NET Framework codebase

- Minimal mentoring or team leadership

- Previously had product/business involvement — wants to get back to that

- Feeling stagnant — no real learning or growth happening



## Goals

- **Primary:** Go deeper technically → Principal Engineer / Architect level

- Learn system design (designing from scratch)

- Learn modern .NET (8/9)

- Go deeper on Azure (architecture level, not just deployment)

- Learn frontend (React or Next.js)

- Eventually build AI-integrated systems (not just use AI as a tool)



## Gaps Identified

1. No system design experience (never designed from scratch)

2. Stuck on legacy .NET Framework — not learning modern .NET

3. Limited Azure depth (surface level)

4. No frontend experience yet

5. No AI integration experience (API level)

6. Limited data modeling knowledge



## Project: Incident Compliance & Reporting Platform



### Domain

EHS (Environment, Health & Safety) software for safety-critical industries:

chemical plants, manufacturing, construction, contractor-based environments.



### Core Domain Concepts

- **Incident** — event, hazard, near-miss, injury. Has lifecycle/status, severity, location, reporter

- **Corrective Action (CA)** — linked to incident, has assignee, status, due date

- **Audit/History Tracking** — full change log: who, what, when, previous vs new values

- **Organizations** — multi-tenant: client orgs + contractors working at client sites



### Core Modules

1. Incident Reporting — create, categorize, severity, location, time

2. Incident Management — status updates, investigation, root cause

3. Corrective Action Management — create, assign, track completion

4. Compliance Repository — historical analysis, pattern detection

5. Audit & Change Tracking — full traceability for regulatory compliance

6. (More modules to emerge as we build)



### Key Architectural Challenges

- Multi-tenancy (client orgs + contractors) — must get right early

- Audit trailing properly (event sourcing concepts)

- State machines (incident lifecycle)

- Role-based access control (safety officer, compliance manager, employee, contractor)

- Scope discipline — phase carefully



### Target Stack

- Backend: .NET 8 Web API (Clean Architecture)

- Frontend: React or Next.js (minimal, functional)

- Database: Azure SQL

- Cloud: Azure App Service / Container Apps

- Auth: Azure AD B2C or similar

- CI/CD: GitHub Actions

- Secrets: Azure Key Vault

- Later: Claude API / OpenAI — pattern detection, incident classification, risk scoring, report generation



### Build Approach

Design first, code second. Principal engineer approach:

1. Bounded contexts

2. Data model + state machines

3. API contract

4. Multi-tenancy strategy

5. Then folder structure and coding



### Learning Milestones

1. [ ] Architecture and system design completed (bounded contexts, data model, API design)

2. [ ] Multi-tenancy strategy decided

3. [ ] .NET 8 solution structure set up (Clean Architecture)

4. [ ] First API endpoints built and tested

5. [ ] Database designed and EF Core configured

6. [ ] Role-based auth implemented

7. [ ] Audit logging implemented

8. [ ] Azure deployment working (CI/CD pipeline)

9. [ ] Frontend connected

10. [ ] AI feature integrated



### Recommended Reading

- *Designing Data-Intensive Applications* — Martin Kleppmann (DDIA)

- System Design Primer (GitHub)

- Microsoft Learn .NET 8 path

- AZ-204 Azure Developer certification (target)



## Session Notes

- First conversation: March 2026

- User wants continuity across sessions — always resume from this file
