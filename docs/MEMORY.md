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

- Current status: Phase 0 complete. Phase 1 COMPLETE. Phase 2 is next.

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

## Phase 1 — COMPLETE (Sprint 2)

### All Stories Done
- EHS-11 ✅ BaseEntity, ITenantEntity, all enums (CompanyType, IncidentType, Severity, IncidentStatus, CorrectiveActionStatus, UserRole)
- EHS-12 ✅ Organization, Site, SiteArea entities
- EHS-13 ✅ Incident entity
- EHS-14 ✅ EF Core 8, ApplicationDbContext, IApplicationDbContext, DependencyInjection, connection string
- EHS-15 ✅ InitialCreate migration — Organizations, Sites, SiteAreas, Incidents tables created in SQL Server
- EHS-16 ✅ MediatR 12.2.0, FluentValidation 11.9.0, CreateIncidentCommand, Handler, Validator, Application DependencyInjection
- EHS-17 ✅ GetIncidentsQuery + GetIncidentByIdQuery handlers, IncidentDto
- EHS-18 ✅ IncidentsController — POST /api/incidents, GET /api/incidents, GET /api/incidents/{id:guid}
- EHS-20 ✅ dev-notes.md written, docs repo updated

### Next Session Starts Here — Phase 2: Incident Lifecycle (Status Machine)
- Incident.TransitionTo(newStatus) domain method
- InvalidStatusTransitionException
- UpdateIncidentCommand, UpdateIncidentStatusCommand, AssignIncidentCommand
- PaginatedList<T> + filters on GetIncidents (status, severity, type, site)

### Local Environment
- SQL Server 2022 local — EHSPlatform database exists with 4 tables
- Connection: Server=localhost;Database=EHSPlatform;Trusted_Connection=True;TrustServerCertificate=True;
- No Docker needed
- Git UI: Fork
- IDE: Visual Studio 2022 Community
- Repo: C:\Projects\ehs-platform
- Jira: pmashroo.atlassian.net — Sprint 2 active

### Session History Update
- Session 4 (Apr 2026): Phase 0 complete. Phase 1 started. EHS-11 through EHS-16 complete.
- Session 5 (Apr 2026): EHS-17, EHS-18, EHS-20 complete. Phase 1 DONE. Ready for Phase 2.
