# EHS Platform — Developer Notes

> Living document. Updated every session. Source of truth for decisions made during actual coding — not planning.

---

## Project at a Glance

| Item | Value |
|---|---|
| Repo | `C:\Projects\ehs-platform` |
| IDE | Visual Studio 2022 Community |
| Git UI | Fork |
| Jira | pmashroo.atlassian.net |
| Active Sprint | Sprint 2 |
| Database | SQL Server 2022 local (no Docker needed) |
| Connection String | `Server=localhost;Database=EHSPlatform;Trusted_Connection=True;TrustServerCertificate=True;` |
| DB Name | EHSPlatform |
| Docs Repo | `C:\Projects\ehs-platform-docs` |
| Claude Code | Primary AI assistant — manages Jira, docs, and code |

---

## Current Phase Status

| Phase | Status |
|---|---|
| Phase 0 — Project Skeleton | ✅ Complete |
| Phase 1 — Core Domain + Incident Reporting | ✅ Complete |
| Phase 2 — Incident Lifecycle (Status Machine) | ⬜ Next |
| Phase 3+ | Not started |

---

## Phase 1 Ticket Status

| Ticket | Description | Status |
|---|---|---|
| EHS-11 | BaseEntity, ITenantEntity, all enums | ✅ Done |
| EHS-12 | Organization, Site, SiteArea entities | ✅ Done |
| EHS-13 | Incident entity | ✅ Done |
| EHS-14 | EF Core 8, ApplicationDbContext, IApplicationDbContext, DI, connection string | ✅ Done |
| EHS-15 | InitialCreate migration — 4 tables created in SQL Server | ✅ Done |
| EHS-16 | MediatR 12.2.0, FluentValidation 11.9.0, CreateIncidentCommand + Handler + Validator, Application DI | ✅ Done |
| EHS-17 | GetIncidentsQuery + Handler, GetIncidentByIdQuery + Handler | ✅ Done |
| EHS-18 | IncidentsController — POST, GET /api/incidents, GET /api/incidents/{id:guid} | ✅ Done |
| EHS-20 | dev-notes.md written, docs repo updated, Phase 1 marked complete | ✅ Done |

---

## Solution Structure

```
EHSPlatform/
├── src/
│   ├── EHSPlatform.Domain/
│   ├── EHSPlatform.Application/
│   ├── EHSPlatform.Infrastructure/
│   └── EHSPlatform.API/
├── tests/
│   ├── EHSPlatform.Domain.Tests/
│   ├── EHSPlatform.Application.Tests/
│   └── EHSPlatform.API.Tests/
├── frontend/               ← Phase 12, not started
└── docker-compose.yml      ← SQL Server + Redis (Redis from Phase 8)
```

**Project references:** API → Infrastructure → Application → Domain. Domain has zero dependencies.

---

## How to Run Locally

```powershell
dotnet run --project src/EHSPlatform.API
```

Open: `http://localhost:5147/swagger`

**Endpoints live in Swagger:**
- `POST /api/incidents` — create incident
- `GET /api/incidents` — list all
- `GET /api/incidents/{id}` — get by Guid

---

## NuGet Packages Installed

| Package | Version | Project | Purpose |
|---|---|---|---|
| MediatR | 12.2.0 | Application | CQRS dispatcher |
| FluentValidation | 11.9.0 | Application | Command validation |
| Microsoft.EntityFrameworkCore | 8.x | Infrastructure | ORM |
| Microsoft.EntityFrameworkCore.SqlServer | 8.x | Infrastructure | SQL Server provider |
| Microsoft.EntityFrameworkCore.Tools | 8.x | Infrastructure | Migrations |

> Update this table every time a new package is added.

---

## Database

### Tables Created (InitialCreate migration)

- `Organizations`
- `Sites`
- `SiteAreas`
- `Incidents`

### Migration Commands

```bash
# Run from solution root
dotnet ef migrations add <MigrationName> --project src/EHSPlatform.Infrastructure --startup-project src/EHSPlatform.API

dotnet ef database update --project src/EHSPlatform.Infrastructure --startup-project src/EHSPlatform.API
```

### Connection String Location

`src/EHSPlatform.API/appsettings.json`

```json
"ConnectionStrings": {
  "DefaultConnection": "Server=localhost;Database=EHSPlatform;Trusted_Connection=True;TrustServerCertificate=True;"
}
```

---

## Domain — Key Entities & Enums

### Enums (all in EHSPlatform.Domain/Enums/)

| Enum | Values |
|---|---|
| `CompanyType` | Client = 1, Contractor = 2 |
| `IncidentType` | Injury, NearMiss, HazardousCondition, EquipmentFailure, EnvironmentalHazard, SafetyViolation, ChemicalSpill, Other |
| `Severity` | Low, Medium, High, Critical |
| `IncidentStatus` | Reported, UnderInvestigation, AwaitingAction, Resolved, Closed |
| `CorrectiveActionStatus` | Open, InProgress, Completed, Verified, Cancelled |
| `UserRole` | OrganizationAdmin, SafetyOfficer, ComplianceManager, IncidentReviewer, SiteEmployee, ContractorAdmin, ContractorSupervisor, ContractorWorker |

### BaseEntity (EHSPlatform.Domain/Common/BaseEntity.cs)

```csharp
public abstract class BaseEntity
{
    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

### ITenantEntity (EHSPlatform.Domain/Common/ITenantEntity.cs)

```csharp
public interface ITenantEntity
{
    Guid TenantId { get; set; }
}
```

> Every domain entity that belongs to a tenant implements this. EF Core Global Query Filters on TenantId added in Phase 5.

---

## Application Layer Patterns

### CQRS Pattern (MediatR)

Every feature gets its own folder under `Application/{Feature}/`:

```
Incidents/
├── Commands/
│   └── CreateIncident/
│       ├── CreateIncidentCommand.cs          ← record + IRequest<Guid>
│       ├── CreateIncidentCommandHandler.cs
│       └── CreateIncidentCommandValidator.cs
└── Queries/
    ├── GetIncidents/
    │   ├── GetIncidentsQuery.cs              ← record + IRequest<List<IncidentDto>>
    │   ├── GetIncidentsQueryHandler.cs
    │   └── IncidentDto.cs
    └── GetIncidentById/
        ├── GetIncidentByIdQuery.cs           ← record(Guid Id) + IRequest<IncidentDto>
        └── GetIncidentByIdQueryHandler.cs
```

### Command Pattern (example)

```csharp
public record CreateIncidentCommand(...) : IRequest<Guid>;

public class CreateIncidentCommandHandler : IRequestHandler<CreateIncidentCommand, Guid>
{
    private readonly IApplicationDbContext _context;
    public CreateIncidentCommandHandler(IApplicationDbContext context) => _context = context;

    public async Task<Guid> Handle(CreateIncidentCommand request, CancellationToken cancellationToken)
    {
        // business logic
        await _context.SaveChangesAsync(cancellationToken);
        return entity.Id;
    }
}
```

### Query Pattern (example)

```csharp
public record GetIncidentsQuery : IRequest<List<IncidentDto>>;

public class GetIncidentsQueryHandler : IRequestHandler<GetIncidentsQuery, List<IncidentDto>>
{
    private readonly IApplicationDbContext _context;

    public async Task<List<IncidentDto>> Handle(GetIncidentsQuery request, CancellationToken cancellationToken)
    {
        return await _context.Incidents
            .Where(x => !x.IsDeleted)
            .OrderByDescending(x => x.CreatedAt)
            .Select(x => new IncidentDto { ... })
            .ToListAsync(cancellationToken);
    }
}
```

### Validator Pattern (FluentValidation)

```csharp
public class CreateIncidentCommandValidator : AbstractValidator<CreateIncidentCommand>
{
    public CreateIncidentCommandValidator()
    {
        RuleFor(x => x.Title).NotEmpty().MaximumLength(200);
        RuleFor(x => x.SiteId).NotEmpty();
        // etc.
    }
}
```

Validators are auto-discovered and wired into the MediatR pipeline via `ValidationBehaviour`.

### IApplicationDbContext (Application/Common/Interfaces/)

```csharp
public interface IApplicationDbContext
{
    DbSet<Incident> Incidents { get; }
    DbSet<Organization> Organizations { get; }
    DbSet<Site> Sites { get; }
    DbSet<SiteArea> SiteAreas { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken);
}
```

> Handlers depend on this interface, NOT on ApplicationDbContext directly. Keeps Application layer clean.

### MediatR Pipeline Behaviours

Order they run in:

```
1. ValidationBehaviour    → FluentValidation on every command
2. LoggingBehaviour       → logs command name + execution time  (Phase 2+)
3. AuditBehaviour         → records writes to AuditLog table    (Phase 6)
→ Handler                 → actual business logic
```

---

## API Layer Patterns

### Controller Pattern

```csharp
[ApiController]
[Route("api/[controller]")]
public class IncidentsController : ControllerBase
{
    private readonly ISender _sender;

    public IncidentsController(ISender sender) => _sender = sender;

    [HttpPost]
    public async Task<IActionResult> Create(CreateIncidentCommand command, CancellationToken cancellationToken)
    {
        var id = await _sender.Send(command, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id }, new { id });
    }

    [HttpGet]
    public async Task<IActionResult> GetAll(CancellationToken cancellationToken)
    {
        var result = await _sender.Send(new GetIncidentsQuery(), cancellationToken);
        return Ok(result);
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        var result = await _sender.Send(new GetIncidentByIdQuery { Id = id }, cancellationToken);
        return Ok(result);
    }
}
```

**Key decisions:**
- `ISender` not `IMediator` — leaner interface, only exposes `Send`
- `CreatedAtAction` on POST — returns HTTP 201 with Location header (correct REST, not just 200)
- `{id:guid}` route constraint — auto-rejects non-Guid ids with 400 before hitting the handler
- Zero business logic in controllers — receive HTTP, send to MediatR, return result

### Error Handling

`ExceptionHandlingMiddleware` catches all exceptions and returns Problem Details (RFC 7807).

| Exception | HTTP Status |
|---|---|
| `NotFoundException` | 404 |
| `ForbiddenException` | 403 |
| `InvalidStatusTransitionException` | 422 |
| `ValidationException` (FluentValidation) | 400 |
| Any other | 500 |

### Middleware Pipeline Order (Program.cs)

```
1. CorrelationIdMiddleware
2. ExceptionHandlingMiddleware
3. Serilog request logging
4. HTTPS redirection
5. Authentication              ← Phase 4
6. Authorization               ← Phase 4
7. TenantResolutionMiddleware  ← Phase 5
8. Controllers
```

---

## Architectural Decisions (Locked — Never Revisit)

| Decision | Choice | Why |
|---|---|---|
| Architecture | Clean Architecture + CQRS + MediatR | Testable, scalable, industry standard |
| Frontend | Single React app, role-based views | One codebase, one deployment |
| Multi-tenancy | Row-level via EF Core Global Query Filters on TenantId | Simple, effective, no schema complexity |
| CompanyType | Client vs Contractor enum on Organization | Drives entire auth model |
| Site hierarchy | Organization → Site → SiteArea | Incidents link to Site, not free text |
| Auth | JWT inside API (no IdentityServer) | Simpler, sufficient for early phases |
| Notifications | Outbox Pattern → email (no RabbitMQ) | Reliable without broker complexity |
| Form engine | Supplementary only — core entities stay typed | Core data must be strongly typed |
| Soft deletes | IsDeleted flag everywhere | No hard deletes, ever |
| Lookup data | DB-driven per tenant for customisable data | Allows tenant customisation without code |

---

## Incident Status State Machine

```
Reported → UnderInvestigation → AwaitingAction → Resolved → Closed

Rules:
- Reported:              Initial state on creation
- UnderInvestigation:    Requires AssignedToId to be set
- AwaitingAction:        Requires RootCause to be filled
- Resolved:              ALL linked CorrectiveActions must be Completed or Verified
- Closed:                Final. Only SafetyOfficer or OrganizationAdmin can close
- Any state → Reported:  "Re-opened" — creates audit entry

InvalidStatusTransitionException thrown for any invalid jump.
```

---

## Lessons Learned / Gotchas

### EF Core
- Always use `.Where(x => !x.IsDeleted)` in queries until Global Query Filters are in place (Phase 5).
- Use `.Select()` to project to DTOs directly in queries — never return raw entities from handlers.
- Projection in `.Select()` is more performant than loading full entity then mapping.

### MediatR
- Use `ISender` (not `IMediator`) in controllers — cleaner, only exposes `Send`.
- `record` types for Commands and Queries — immutable by default, cleaner syntax.
- Pass `CancellationToken` through to `_sender.Send()` — supports request cancellation.

### FluentValidation
- Validators are auto-registered via DI extension — don't manually register each one.
- `ValidationBehaviour` in the pipeline ensures every command is validated before the handler runs.

### General
- `NotFoundException(nameof(Entity), key)` — always use `nameof()` so refactoring doesn't break error messages.
- Never throw from a Query handler except `NotFoundException`. Queries should never have side effects.
- No auth yet — TenantId and UserId are hardcoded placeholders until Phase 4.

---

## What's Coming (Don't Build Early)

| Phase | What | When |
|---|---|---|
| Phase 2 | Incident status machine, pagination, filtering | Next |
| Phase 3 | Corrective Actions | After Phase 2 |
| Phase 4 | Users, JWT auth, roles | After Phase 3 |
| Phase 5 | Multi-tenancy (Global Query Filters) | After Phase 4 |
| Phase 6 | Audit logging (EF interceptor) | After Phase 5 |
| Phase 7 | Dynamic Form Engine | After Phase 6 |
| Phase 8 | Redis caching | After Phase 7 |
| Phase 9 | Outbox pattern + background workers | After Phase 8 |
| Phase 10 | File attachments (Azure Blob) | After Phase 9 |
| Phase 11 | Dashboard & reporting | After Phase 10 |
| Phase 12 | React frontend | After Phase 11 |
| Phase 13 | Work Permit Management | Later |
| Phase 14 | Document Management | Later |
| Phase 15 | Compliance Audit Management | Later |
| Phase 16 | Azure deployment + CI/CD | Later |
| Phase 17 | AI integration (Claude API) | Later |

---

## Anti-Procrastination Rules (Read This If Drifting)

1. One phase at a time. Never plan Phase 5 while building Phase 1.
2. Each session ends with working, committed code. No half-finished features.
3. No gold-plating early phases. Phase 1 is intentionally ugly. That is correct.
4. Decisions are made once. Never revisit technology choices.
5. Done is better than perfect.
6. Minimum viable phase — if a phase feels too big, cut it in half.
7. No rabbit holes — new ideas go in a future phase note. Not now.

---

## Session Log

| Session | Date | What Happened |
|---|---|---|
| Session 1 | Mar 2026 | Defined career goals, initial project concept |
| Session 2 | Mar 2026 | MAISAAS deep analysis, full EHS scope, all architecture decisions locked |
| Session 3 | Mar 2026 | MAISAAS deep-analysis doc generated. MAISAAS deleted from company laptop. |
| Session 4 | Apr 2026 | Phase 0 complete. Phase 1 started. EHS-11 through EHS-16 complete. |
| Session 5 | Apr 2026 | EHS-17, EHS-18, EHS-20 complete. Phase 1 DONE. GitHub + Jira connected to Claude Code. Docs repo cloned locally. Ready for Phase 2. |

---

*Update this file at the end of every session. Commit it with the code.*
