# EHS Platform — Developer Notes

> Project vitals only. For everything else, see the router in START_HERE.md.

---

## Project at a Glance

| Item | Value |
|---|---|
| Repo | `C:\Projects\ehs-platform` |
| IDE | Visual Studio 2022 Community |
| Git UI | Fork |
| Jira | pmashroo.atlassian.net — Sprint 8 |
| Database | SQL Server 2022 local (no Docker needed) |
| Connection String | `Server=localhost;Database=EHSPlatform;Trusted_Connection=True;TrustServerCertificate=True;` |
| Docs Repo | `C:\Projects\ehs-platform-docs` |

---

## Current Phase Status

| Phase | Status |
|---|---|
| Phase 0 — Project Skeleton | ✅ Complete |
| Phase 1 — Core Domain + Incident Reporting | ✅ Complete |
| Phase 2 — Incident Lifecycle (Status Machine) | ✅ Complete |
| Phase 3 — Corrective Actions | ✅ Complete |
| Phase 4 — Users & Authentication | ✅ Complete |
| Phase 5 — Multi-tenancy + Infrastructure Plumbing | ✅ Complete |
| Phase 6 — Audit Logging | ✅ Complete |
| Phase 7 — Hardening & Multi-Reviewer Audit | 🔄 In Progress |

**Active tickets → see `docs/current/phase-7-status.md`**

---

## Solution Structure

```
EHSPlatform/
├── src/
│   ├── EHSPlatform.Domain/
│   ├── EHSPlatform.Application/
│   ├── EHSPlatform.Infrastructure/
│   └── EHSPlatform.API/
│       ├── Controllers/
│       ├── Middleware/
│       └── Converters/
├── tests/
│   ├── EHSPlatform.Domain.Tests/
│   ├── EHSPlatform.Application.Tests/
│   └── EHSPlatform.Infrastructure.Tests/
└── docker-compose.yml   ← Redis + Elasticsearch 8.13.4 + Kibana (SQL Server runs locally)
```

Run: `dotnet run --project src/EHSPlatform.API` → `http://localhost:5147/swagger`

---

## NuGet Packages

| Package | Version | Project | Purpose |
|---|---|---|---|
| MediatR | 12.2.0 | Application | CQRS dispatcher |
| FluentValidation | 11.9.0 | Application | Command validation |
| Microsoft.EntityFrameworkCore | 8.x | Infrastructure | ORM |
| Microsoft.EntityFrameworkCore.SqlServer | 8.x | Infrastructure | SQL Server provider |
| Microsoft.EntityFrameworkCore.Tools | 8.x | Infrastructure | Migrations |
| Microsoft.EntityFrameworkCore.InMemory | 8.0.0 | Tests | In-memory DB — pin to 8.0.0 |
| xUnit | 2.x | All Tests | Unit test framework |
| FluentAssertions | 8.9.0 | All Tests | Assertion library |
| Moq | 4.20.72 | Application.Tests | Mocking |
| BCrypt.Net-Next | 4.0.3 | Infrastructure | Password hashing |
| Microsoft.AspNetCore.Authentication.JwtBearer | 8.0.0 | Infrastructure | JWT Bearer |

> Add new packages to this table every time one is installed.

---

## Database

**Migration commands (run from solution root):**
```bash
dotnet ef migrations add <Name> --project src/EHSPlatform.Infrastructure --startup-project src/EHSPlatform.API
dotnet ef database update --project src/EHSPlatform.Infrastructure --startup-project src/EHSPlatform.API
```

**Connection string location:** `src/EHSPlatform.API/appsettings.json`

---

## Middleware Pipeline Order

```
1. CorrelationIdMiddleware
2. ExceptionHandlingMiddleware
3. Serilog request logging
4. HTTPS redirection
5. Authentication (JWT)
6. Authorization
7. TenantResolutionMiddleware
8. Controllers
```

---

## Error Handling — HTTP Status Map

| Exception | HTTP Status |
|---|---|
| `NotFoundException` | 404 |
| `ForbiddenException` | 403 |
| `InvalidStatusTransitionException` | 422 |
| `ValidationException` (FluentValidation) | 400 |
| `DbUpdateConcurrencyException` | 409 |
| Any other | 500 |

---

## Architectural Decisions (Locked)

| Decision | Choice |
|---|---|
| Architecture | Clean Architecture + CQRS + MediatR |
| Multi-tenancy | Row-level via EF Core Global Query Filters on TenantId |
| Auth | JWT inside API (no IdentityServer) |
| Soft deletes | IsDeleted flag everywhere — no hard deletes ever |
| Timestamps | `DateTimeOffset` everywhere — `datetimeoffset(7)` in SQL Server (ADR-016) |
| Notifications | Outbox Pattern → email (Phase 9) |
| Caching | `ICacheService` over `IDistributedCache` (Redis) — Dapr State candidate at Phase 8 |
| Search | Elasticsearch (Phase 11) — SQL FTS until then |

**Full ADRs → `docs/architecture-decisions.md`**
