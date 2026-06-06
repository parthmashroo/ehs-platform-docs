# Memory Index



## Active Files

- [dev-notes.md](dev-notes.md) — **Primary source of truth.** Ticket-by-ticket status, session log, code patterns, lessons learned
- [technical-debt.md](technical-debt.md) — Architecture review findings, severity, target phase, Jira ticket ref
- [architecture-decisions.md](architecture-decisions.md) — All ADRs (locked — never revisit)
- [project-roadmap.md](project-roadmap.md) — Full technical blueprint, all 17 phases, decisions, full scope
- [career-profile.md](career-profile.md) — User's engineering background, goals, and learning plan
- [architectural-gaps.md](architectural-gaps.md) — 25 architectural gaps reviewed in Session 16, decisions recorded
- [ai-first-strategy.md](ai-first-strategy.md) — Phase 17 AI architecture vision — MCP, AI Service Principal
- [ai-capabilities-research.md](ai-capabilities-research.md) — Market research, competitor analysis
- [semantic-form-engine-design.md](semantic-form-engine-design.md) — Phase 7 form engine — semantic AI layer, scoring, pre-built templates



## MAISAAS Reference (Original System — Code Has Been Deleted)

The MAISAAS project that inspired the EHS platform no longer exists on disk.

A full deep-analysis document was generated before deletion. It covers all 20 technical areas:

solution structure, auth, data access, service layer, email, jobs, messaging, multi-tenancy,

questionnaire module, work permit module, all entities, DI, error handling, frontend patterns.



**Original location (may have changed — verify before reading):**
`C:\Users\Komal\.claude\projects\...\memory\maisaas-deep-analysis.md`

Also stored at: `C:\Projects\ehs-platform-docs\docs\maisaas-deep-analysis.md`

When discussing MAISAAS patterns or comparing to the new build, read that file first.



## Quick Reference

- User: .NET backend engineer, 11+ years, wants Principal/Architect level
- Project: **EHS Platform** (Environment, Health & Safety) — full SaaS, not just incident reporting
- Inspired by: MAISAAS (real ERP system the user worked on 2-3 years ago)
- Practice machine: Personal machine only — NOT the company laptop
- Current status: Phase 0 ✅ Phase 1 ✅ Phase 2 ✅ Phase 3 ✅ Phase 4 ✅ Phase 5 ✅ Phase 6 ✅ Phase 7 🔄 IN PROGRESS (EHS-69/65/66 done, Sprint 8)
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
- Caching: IDistributedCache + ICacheService abstraction over Redis — FusionCache rejected (ADR decision)
- Search: Elasticsearch for full-text (local Docker/Podman), not SQL Full-Text Search
- Dapr: Deferred to Phase 8 as a candidate anchor — State Store as L2 cache swap, not locked
- Concurrency: RowVersion on all BaseEntity types, ETag/If-Match on API (ADR-012)
- Audit logging: EF SaveChangesInterceptor writes AuditLog rows — not MediatR pipeline behaviour



## Local Environment

- SQL Server 2022 local — EHSPlatform database
- Connection: `Server=localhost;Database=EHSPlatform;Trusted_Connection=True;TrustServerCertificate=True;`
- Redis: Podman Desktop on WSL2 (redis:7.2-alpine via docker-compose.yml)
- Elasticsearch 8.13.4 + Kibana 8.13.4: Podman via docker-compose.yml at solution root
- No Docker needed for SQL Server
- Git UI: Fork
- IDE: Visual Studio 2022 Community
- Repo: `C:\Projects\ehs-platform`
- Docs repo: `C:\Projects\ehs-platform-docs`
- Jira: pmashroo.atlassian.net — Sprint 8 active



## Phase Status

| Phase | Status | Sprint |
|---|---|---|
| Phase 0 — Project Skeleton | ✅ Complete | Sprint 1 |
| Phase 1 — Core Domain + Incident Reporting | ✅ Complete | Sprint 2 |
| Phase 2 — Incident Lifecycle (State Machine) | ✅ Complete | Sprint 3 |
| Phase 3 — Corrective Actions | ✅ Complete | Sprint 3 |
| Phase 4 — Users & Authentication | ✅ Complete | Sprint 5–6 |
| Phase 5 — Multi-tenancy + Infrastructure Plumbing | ✅ Complete | Sprint 6–7 |
| Phase 6 — Audit Logging | ✅ Complete | Sprint 7 |
| Phase 7 — Hardening & Multi-Reviewer Audit | 🔄 In Progress | Sprint 8 |



## Session History (summary — full log in dev-notes.md)

- Session 1–3 (Mar 2026): Career goals defined, MAISAAS analysed, architecture locked, MAISAAS deleted from company laptop
- Session 4–5 (Apr 2026): Phase 0 + Phase 1 DONE (EHS-11 to EHS-20)
- Session 6 (Apr 2026): Phase 2 DONE — state machine, pagination, all PATCH/PUT endpoints
- Session 7–8 (May 2026): Phase 3 DONE (corrective actions). Phase 4 started — User entity, ICurrentUserService, AuthController + JWT + BCrypt, 27 tests
- Session 9 (May 2026): Phase 4 DONE — [Authorize] on all endpoints, role guards, ForbiddenAccessException → 403, 37 tests green
- Session 10 (May 2026): EHS-45 Architecture Spike DONE — ADR-012 written (RowVersion, ETag, audit tables, Redis, Elasticsearch). Sprint 3 closed, Sprint 4 started.
- Session 11 (May 2026): Carried-over fixes — EHS-35, EHS-36, EHS-44. Error messages, FK constraints, role guard on re-open.
- Session 12 (May 2026): EHS-46 DONE — RowVersion on BaseEntity, .IsRowVersion() on all 6 EF configs, 409 on concurrency conflict, 45 tests green
- Session 13 (May 2026): EHS-47 DONE — CorrelationId middleware, IRequestContext, Serilog enrichment
- Session 14 (May 2026): EHS-48 DONE — TenantId on all entities, combined migration. AI-first strategy docs written.
- Session 15 (May 2026): EHS-49 DONE (ClientContractor entity). EHS-50 DONE — IsDeleted on Site/SiteArea/Org/User, Global Query Filters, TenantResolutionMiddleware, 46 tests green
- Session 16 (May 2026): Architectural gap review — 25 gaps, Dapr deferred, ICacheService scope revised, CORS fix + NetArchTest identified
- Session 17 (May 2026): EHS-51 DONE — ICacheService + DistributedCacheService + Redis + Infrastructure.Tests project (3 tests)
- Session 18 (May 2026): EHS-52 DONE — docker-compose.yml, Elasticsearch + Kibana via Podman, full-text search spike vs SQL Full-Text
- Session 19 (May 2026): EHS-53 DONE — Phase 5 closed out, all docs updated
- Session 20 (May 2026): EHS-56 DONE — AuditLog entity + AuditAction enum + AuditLogConfiguration + migration. 49 tests green. Phase 6 in progress.
- Sessions 21–26 (May–Jun 2026): Phase 6 complete (EHS-57/62/58/59). Phase 7 started — EHS-69 CORS, EHS-65 IAuditableEntity. 62 tests green.
- Session 27 (Jun 2026): EHS-66 DONE — AuditInterceptor TenantId == Guid.Empty guard. 63 tests green.
