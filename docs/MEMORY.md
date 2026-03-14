# Memory Index



## Active Files

- [career-profile.md](https://github.com/parthmashroo/ehs-platform-docs/blob/main/docs/career-profile.md) — User's engineering background, goals, and learning plan

- [project-roadmap.md](https://github.com/parthmashroo/ehs-platform-docs/blob/main/docs/project-roadmap.md) — Full technical blueprint, ALL decisions, all phases



## MAISAAS Reference (Original System — Code Has Been Deleted)

The MAISAAS project that inspired the EHS platform no longer exists on disk.

A full deep-analysis document was generated before deletion. It covers all 20 technical areas:

solution structure, auth, data access, service layer, email, jobs, messaging, multi-tenancy,

questionnaire module, work permit module, all entities, DI, error handling, frontend patterns.



**Location:** `C:\Users\parth.mashroo\.claude\projects\c--Users-parth-mashroo-Downloads-MAISAAS-new-MAISAAS\memory\maisaas-deep-analysis.md`



When discussing MAISAAS patterns or comparing to the new build, read that file first.



## Quick Reference

- User: .NET backend engineer, 11+ years, wants Principal/Architect level

- Project: **EHS Platform** (Environment, Health & Safety) — full SaaS, not just incident reporting

- Inspired by: MAISAAS (real ERP system the user worked on 2-3 years ago)

- Practice machine: Personal machine only — NOT the company laptop

- Current status: **Phase 0 not started. No code written yet.**

- User personality: Self-identified procrastinator and overthinker. Perfectionist. Needs to be kept on task.



## Decisions Made (Never Revisit These)

- Stack: .NET 8 Clean Architecture + CQRS + MediatR + React 18 + TypeScript

- Portal: Single React app, role-based views (NOT two separate apps)

- Form engine: Supplementary use only — core entities stay typed (Incident, CA, etc.)

- ISO templates: Seed a few in Phase 7 to prove the engine, commercial expansion later

- Notifications: Outbox Pattern (not RabbitMQ) for now, Azure Service Bus later

- Auth: JWT in-API (no separate IdentityServer), SCIM/SSO in a later phase

- Multi-tenancy: Row-level isolation via EF Core Global Query Filters on TenantId

- CompanyType: Client (site owner) vs Contractor — fundamental distinction

- Site hierarchy: Organization → Site → SiteArea (incidents linked to sites, not free text)

- Lookup tables: Some domain data (IncidentType categories, Area names) is DB-driven per tenant



## Session History

- Session 1 (Mar 2026): Defined career goals, initial project concept

- Session 2 (Mar 2026): Deep analysis of MAISAAS, full EHS scope decided, all architecture decisions locked

- Session 3 (Mar 2026): MAISAAS deep-analysis document generated and saved. MAISAAS project deleted from company laptop. Ready to start Phase 0 on personal machine. 
