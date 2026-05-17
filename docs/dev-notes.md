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
| Active Sprint | Sprint 3 |
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
| Phase 2 — Incident Lifecycle (Status Machine) | ✅ Complete |
| Phase 3 — Corrective Actions | ✅ Complete |
| Phase 4 — Users & Authentication | ✅ Complete |
| Phase 5+ | Not started |

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

## Phase 2 Ticket Status

| Ticket | Description | Status |
|---|---|---|
| EHS-21 | Incident.TransitionTo() domain method + InvalidStatusTransitionException | ✅ Done |
| EHS-22 | UpdateIncidentCommand, UpdateIncidentStatusCommand, AssignIncidentCommand + handlers + validators | ✅ Done |
| EHS-23 | PaginatedList\<T\>, GetIncidentsQuery with pagination and filters | ✅ Done |
| EHS-24 | PUT, PATCH /status, PATCH /assign endpoints in IncidentsController | ✅ Done |
| EHS-25 | Docs updated for Phase 2 completion | ✅ Done |

---

## Phase 3 Ticket Status

| Ticket | Description | Status |
|---|---|---|
| EHS-26 | CorrectiveAction entity, EF config, AddCorrectiveActions migration | ✅ Done |
| EHS-27 | CreateCorrectiveActionCommand + queries + controller (GET list, GET by id) + unit tests for Incident.TransitionTo() | ✅ Done |
| EHS-28 | CorrectiveAction.TransitionTo() domain method + UpdateCorrectiveAction + UpdateCorrectiveActionStatus commands + unit tests | ✅ Done |
| EHS-29 | SoftDeleteCorrectiveActionCommand + DELETE endpoint | ✅ Done |
| EHS-30 | (merged into EHS-29) | ✅ Done |
| EHS-31 | Docs updated for Phase 3 in-progress state | ✅ Done |
| — | Business rule: Incident → Resolved only if all CAs Completed/Verified | ✅ Done |
| — | GetIncidentByIdQuery to include linked CAs in response | ✅ Done |

---

## Phase 4 Ticket Status

| Ticket | Description | Status |
|---|---|---|
| EHS-39 | User entity, UserRole enum, EF config, migration | ✅ Done |
| EHS-40 | ICurrentUserService interface + implementation | ✅ Done |
| EHS-41 | AuthController — Register + Login + JWT generation + global email uniqueness + IsActive login guard | ✅ Done |
| EHS-42 | [Authorize] on all endpoints + role restrictions + wire ICurrentUserService | ✅ Done |
| EHS-43 | Phase 4 docs update | ✅ Done |
| EHS-45 | Architecture Spike — Concurrency, Consistency, Auditability & Traceability | ✅ Done |
| EHS-35 | FK constraints for ReportedById/AssignedToId + re-open from Closed role guard | ✅ Done |
| EHS-36 | PATCH /assign returns 200 with updated status instead of 204 | ✅ Done |
| EHS-44 | Fix user-facing error messages — DomainValidationException, entity-agnostic InvalidStatusTransitionException, ToDisplayName() extensions, plain-language NotFoundException, UpdateCorrectiveAction 500 → 422 | ✅ Done |

---

## Phase 5 Ticket Status

| Ticket | Description | Status |
|---|---|---|
| EHS-46 | RowVersion on BaseEntity + .IsRowVersion() on all 6 EF configs + DbUpdateConcurrencyException → 409 + migration + concurrency test | ✅ Done |
| EHS-47 | CorrelationId: Serilog enricher + IRequestContext + stamp on all audit rows | ✅ Done |
| EHS-48 | TenantId on all existing entities + combined migration (with EHS-46 RowVersion) | ✅ Done |
| EHS-49 | ClientContractor entity + EF config + migration | ✅ Done |
| EHS-50 | EF Core Global Query Filters + TenantResolutionMiddleware | ✅ Done |
| EHS-51 | Redis + FusionCache: docker-compose/Podman wiring + cache key conventions | ⬜ To Do |
| EHS-52 | Elasticsearch + Kibana local Docker/Podman setup + SQL Full-Text comparison spike | ⬜ To Do |
| EHS-53 | Phase 5 docs update | ⬜ To Do |

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
- `POST /api/incidents` — report a new incident
- `GET /api/incidents` — get a list of incidents (supports `?pageNumber=1&pageSize=10&status=&severity=&siteId=`)
- `GET /api/incidents/{id}` — get incident details
- `PUT /api/incidents/{id}` — update incident information
- `PATCH /api/incidents/{id}/status` — change the status of an incident
- `PATCH /api/incidents/{id}/assign` — assign an investigator to the incident
- `POST /api/correctiveactions` — log a new corrective action
- `GET /api/correctiveactions` — get a list of corrective actions (supports `?incidentId=&status=&pageNumber=1&pageSize=10`)
- `GET /api/correctiveactions/{id}` — get corrective action details
- `PUT /api/correctiveactions/{id}` — update a corrective action
- `PATCH /api/correctiveactions/{id}/status` — update corrective action status
- `POST /api/auth/register` — create a new organisation and first admin user, returns JWT
- `POST /api/auth/login` — login with email and password, returns JWT

---

## NuGet Packages Installed

| Package | Version | Project | Purpose |
|---|---|---|---|
| MediatR | 12.2.0 | Application | CQRS dispatcher |
| FluentValidation | 11.9.0 | Application | Command validation |
| Microsoft.EntityFrameworkCore | 8.x | Infrastructure | ORM |
| Microsoft.EntityFrameworkCore.SqlServer | 8.x | Infrastructure | SQL Server provider |
| Microsoft.EntityFrameworkCore.Tools | 8.x | Infrastructure | Migrations |
| xUnit | 2.x | Domain.Tests, Application.Tests | Unit test framework |
| FluentAssertions | 8.9.0 | Domain.Tests, Application.Tests | Assertion library (non-commercial only — see technical-debt.md) |
| BCrypt.Net-Next | 4.0.3 | Infrastructure | Password hashing (salt embedded in hash — no PasswordSalt column) |
| Microsoft.AspNetCore.Authentication.JwtBearer | 8.0.0 | Infrastructure | JWT Bearer middleware + token validation |
| Moq | 4.20.72 | Application.Tests | Mocking interfaces in handler tests |
| Microsoft.EntityFrameworkCore.InMemory | 8.0.0 | Application.Tests | In-memory DB for handler integration tests |

> **Note:** `EHSPlatform.Infrastructure` uses `<FrameworkReference Include="Microsoft.AspNetCore.App" />` in its `.csproj` instead of a NuGet package for ASP.NET Core types (`IHttpContextAccessor`, etc.). This is the correct approach for class libraries targeting `net8.0` — no version number needed, picks up the app's framework version automatically.

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
| `Priority` | Low = 1, Medium = 2, High = 3, Critical = 4 |
| `UserRole` | OrganizationAdmin, SafetyOfficer, ComplianceManager, IncidentReviewer, SiteEmployee, ContractorAdmin, ContractorSupervisor, ContractorWorker |

### BaseEntity (EHSPlatform.Domain/Common/BaseEntity.cs)

```csharp
public abstract class BaseEntity
{
    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}
```

> `RowVersion` is a SQL Server `timestamp`/`rowversion` column — auto-updated by the DB on every write. All 6 entity EF configurations call `.IsRowVersion()`. EF Core adds `WHERE RowVersion = @original` to every UPDATE/DELETE, throwing `DbUpdateConcurrencyException` if another user modified the record first.

### ITenantEntity (EHSPlatform.Domain/Common/ITenantEntity.cs)

```csharp
public interface ITenantEntity
{
    Guid TenantId { get; set; }
}
```

> Every domain entity that belongs to a tenant implements this. EF Core Global Query Filters on TenantId added in Phase 5.

---

## New Entity Checklist — Follow This Every Time

Every new entity in this system touches all 4 layers in a predictable sequence.
Follow this order every time. Never skip steps.

---

### Step 1 — Domain: Enums
`EHSPlatform.Domain/Enums/`

Create one file per enum needed by the entity.
Always assign explicit integer values starting at 1.

```csharp
public enum MyStatusEnum
{
    Active = 1,
    Inactive = 2,
    Archived = 3
}
```

---

### Step 2 — Domain: Entity
`EHSPlatform.Domain/Entities/MyEntity.cs`

Always extend `BaseEntity` (gives Id, CreatedAt, UpdatedAt).
String properties default to `string.Empty`. Booleans default explicitly.
Status property defaults to its initial state.

```csharp
public class MyEntity : BaseEntity
{
    public string Title { get; set; } = string.Empty;
    public MyStatusEnum Status { get; set; } = MyStatusEnum.Active;
    public bool IsDeleted { get; set; } = false;
    // FK to parent entity (nullable if optional link — see ADR-011)
    public Guid? ParentEntityId { get; set; }
}
```

If the entity has business rules on state changes — add a domain method here (like `TransitionTo()`).
Never put business rules in handlers if they belong in the entity.

---

### Step 3 — Infrastructure: EF Configuration
`EHSPlatform.Infrastructure/Persistence/Configurations/MyEntityConfiguration.cs`

One file per entity. Implement `IEntityTypeConfiguration<MyEntity>`.
EF auto-discovers all configurations via `ApplyConfigurationsFromAssembly()` — no manual registration needed.

```csharp
public class MyEntityConfiguration : IEntityTypeConfiguration<MyEntity>
{
    public void Configure(EntityTypeBuilder<MyEntity> builder)
    {
        builder.ToTable("MyEntities");
        builder.HasKey(x => x.Id);

        builder.Property(x => x.Title).IsRequired().HasMaxLength(500);
        builder.Property(x => x.Status).IsRequired();
        builder.Property(x => x.IsDeleted).IsRequired().HasDefaultValue(false);
        builder.Property(x => x.CreatedAt).IsRequired();
        builder.Property(x => x.UpdatedAt).IsRequired();

        // FK relationship (if applicable)
        builder.HasOne<ParentEntity>()
            .WithMany()
            .HasForeignKey(x => x.ParentEntityId)
            .OnDelete(DeleteBehavior.Restrict);

        // Filtered index for nullable FK (ADR-011)
        builder.HasIndex(x => x.ParentEntityId)
            .HasFilter("[ParentEntityId] IS NOT NULL");

        // Soft delete filter — hides deleted records from all queries
        builder.HasQueryFilter(x => !x.IsDeleted);
    }
}
```

---

### Step 4 — Infrastructure: Register DbSet
Two files to update:

**`ApplicationDbContext.cs`** — add the DbSet:
```csharp
public DbSet<MyEntity> MyEntities => Set<MyEntity>();
```

**`IApplicationDbContext.cs`** — add the interface property:
```csharp
DbSet<MyEntity> MyEntities { get; }
```

Both must be updated together. If you update one and forget the other, it will not compile.

---

### Step 5 — Infrastructure: Migration
Run from solution root. Always name migrations descriptively.

```bash
dotnet ef migrations add AddMyEntity \
  --project src/EHSPlatform.Infrastructure \
  --startup-project src/EHSPlatform.API

dotnet ef database update \
  --project src/EHSPlatform.Infrastructure \
  --startup-project src/EHSPlatform.API
```

Review the generated migration file before running update. Check:
- Correct table name
- Correct column types and nullability
- FK constraint present
- Filtered index present

---

### Step 6 — Application: DTO
`EHSPlatform.Application/MyEntities/Queries/MyEntityDto.cs`

What the API returns — never return raw entities from handlers.
Only include fields that callers actually need.

```csharp
public class MyEntityDto
{
    public Guid Id { get; init; }
    public string Title { get; init; } = string.Empty;
    public string Status { get; init; } = string.Empty;
    public DateTime CreatedAt { get; init; }
}
```

---

### Step 7 — Application: Commands (Writes)
`EHSPlatform.Application/MyEntities/Commands/CreateMyEntity/`

Three files per command — always:

**Command** (`CreateMyEntityCommand.cs`) — the input:
```csharp
public record CreateMyEntityCommand : IRequest<Guid>
{
    public string Title { get; init; } = string.Empty;
}
```

**Handler** (`CreateMyEntityCommandHandler.cs`) — the logic:
```csharp
public class CreateMyEntityCommandHandler : IRequestHandler<CreateMyEntityCommand, Guid>
{
    private readonly IApplicationDbContext _context;
    public CreateMyEntityCommandHandler(IApplicationDbContext context) => _context = context;

    public async Task<Guid> Handle(CreateMyEntityCommand request, CancellationToken ct)
    {
        var entity = new MyEntity { Title = request.Title, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow };
        _context.MyEntities.Add(entity);
        await _context.SaveChangesAsync(ct);
        return entity.Id;
    }
}
```

**Validator** (`CreateMyEntityCommandValidator.cs`) — the rules:
```csharp
public class CreateMyEntityCommandValidator : AbstractValidator<CreateMyEntityCommand>
{
    public CreateMyEntityCommandValidator()
    {
        RuleFor(x => x.Title).NotEmpty().MaximumLength(500);
    }
}
```

Repeat this pattern for each write operation (Update, Delete/soft-delete, status change).

---

### Step 8 — Application: Queries (Reads)
`EHSPlatform.Application/MyEntities/Queries/GetMyEntities/`

**Query** (`GetMyEntitiesQuery.cs`):
```csharp
public class GetMyEntitiesQuery : IRequest<PaginatedList<MyEntityDto>>
{
    public int PageNumber { get; init; } = 1;
    public int PageSize { get; init; } = 10;
    // Add nullable filters as needed
}
```

**Handler** (`GetMyEntitiesQueryHandler.cs`):
```csharp
public async Task<PaginatedList<MyEntityDto>> Handle(GetMyEntitiesQuery request, CancellationToken ct)
{
    var safePageSize = Math.Min(request.PageSize, PaginationConstants.MaxPageSize); // always cap — never hardcode the number
    var query = _context.MyEntities.Where(x => !x.IsDeleted);
    var totalCount = await query.CountAsync(ct);
    var items = await query
        .OrderByDescending(x => x.CreatedAt).ThenBy(x => x.Id) // always stable sort
        .Skip((request.PageNumber - 1) * safePageSize)
        .Take(safePageSize)
        .Select(x => new MyEntityDto { Id = x.Id, Title = x.Title, Status = x.Status.ToString() })
        .ToListAsync(ct);
    return new PaginatedList<MyEntityDto>(items, totalCount, request.PageNumber, safePageSize);
}
```

---

### Step 9 — API: Controller
`EHSPlatform.API/Controllers/MyEntitiesController.cs`

```csharp
[ApiController]
[Route("api/[controller]")]
public class MyEntitiesController : ControllerBase
{
    private readonly ISender _sender;
    public MyEntitiesController(ISender sender) => _sender = sender;

    [HttpPost]
    [EndpointSummary("...plain language description for end users...")]
    public async Task<IActionResult> Create(CreateMyEntityCommand command, CancellationToken ct)
    {
        var id = await _sender.Send(command, ct);
        return CreatedAtAction(nameof(GetById), new { id }, new { id });
    }
}
```

Rules:
- `[EndpointSummary]` on every endpoint — plain language, no developer jargon
- `ISender` not `IMediator`
- POST returns 201 `CreatedAtAction`, PUT/PATCH returns 204, GET returns 200
- Zero business logic in controllers

---

### Step 10 — Tests
`tests/EHSPlatform.Domain.Tests/` and `tests/EHSPlatform.Application.Tests/`

Minimum required per entity:
- Unit test: any domain method with rules (like TransitionTo, AssignTo) — test every valid and invalid path
- Integration test: happy path for Create + Get, and one failure case (not found, invalid input)

Tests ship in the same commit as the code. Never deferred.

---

### Step 11 — Docs & Jira
After all code is committed:
1. Update `dev-notes.md` — add to Phase ticket status table
2. Update `project-roadmap.md` — check off phase items if applicable
3. Commit docs to `ehs-platform-docs`
4. Transition Jira ticket → Done

---

## Application Layer Patterns

### CQRS Pattern (MediatR)

Every feature gets its own folder under `Application/{Feature}/`:

#### One command = one intent = one folder. Always.

Each command gets exactly 3 files — no exceptions:
- `XxxCommand.cs` — the input (what the caller sends)
- `XxxCommandHandler.cs` — the logic (fetch → call domain → save)
- `XxxCommandValidator.cs` — the rules (FluentValidation)

**Never put two commands in one file.** If you're tempted to, you're either:
1. Looking at two separate operations that need two folders, or
2. Looking at one operation that needs an enriched command with an optional field + domain guard

**One command = one handler** is a hard rule. The handler is always 1:1 with the command.

#### CQRS at scale — what to expect

By the time this product matures across 17+ phases, expect ~215 command/query folders total:

| Phase group | Estimated operations |
|---|---|
| Phase 1-3 (current) | ~25 |
| Phase 4-8 (auth, tenancy, audit, forms, cache) | ~60 more |
| Phase 9-13 (permits, training, inspections) | ~80 more |
| Phase 14-17 (reporting, mobile, integrations) | ~50 more |
| **Total** | **~215** |

This is healthy, not bloated. Each folder is isolated, testable, and findable in 2 keystrokes (`Ctrl+T`). The alternative — service classes with many methods — becomes a 3000-line maintenance nightmare by year 2. The folder count is a sign of health: every intent has one home, one handler, one test surface.

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
    DbSet<CorrectiveAction> CorrectiveActions { get; }
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
| `DbUpdateConcurrencyException` | 409 |
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

## Design Q&A — Decisions Captured During Development

### Soft-deleted user with active JWT — known gap (fix in Phase 8)

**Q:** If a user is soft-deleted, can they still use their existing JWT?

**A:** Yes — for up to 12 hours (current JWT expiry). The `HasQueryFilter(x => !x.IsDeleted)` on `User` blocks new logins immediately. But a JWT already in the client's hands is cryptographically valid until it expires. `TenantResolutionMiddleware` only checks for the `tid` claim — it does not validate that the user still exists or is not deleted.

**Why it's hard to catch later:** Requires two concurrent sessions to reproduce. The window is up to 12 hours and leaves no error in logs — access just silently continues.

**Fix:** Phase 8 (Redis). When a user is soft-deleted, write their JWT `jti` claim to a Redis blocklist with TTL = remaining token expiry. `TenantResolutionMiddleware` checks the blocklist on every authenticated request. Requires adding `jti` claim to `LoginCommandHandler` first.

| State | Behaviour |
|---|---|
| Deleted user attempts new login | Blocked — query filter returns no user |
| Deleted user uses existing JWT | Allowed until expiry — **gap** |
| After Phase 8 blocklist | Blocked immediately on deletion |

Tracked as Jira ticket — resolve in Phase 8 alongside Redis introduction.

---

### CorrelationId scope — why not link across requests?

**Q:** If a user clicks something and nothing happens, then clicks again and gets an error — both have different CorrelationIds. How do you get the full picture?

**A:** CorrelationId is per-request scope only. The full user journey is solved by the Phase 6 AuditLog, not by chaining CorrelationIds. The audit table records every write by `UserId` in order — each row carries its own CorrelationId as a link to that request's logs. Filter audit rows by `UserId` for the journey; use CorrelationId to drill into any single request's logs.

| Concern | Tool |
|---|---|
| Single request trace | CorrelationId (EHS-47) |
| User journey over time | AuditLog (Phase 6) |
| Cross-service trace | OpenTelemetry (future) |

### Does logging/auditing block the request?

**Q:** Should logging and auditing run in a separate service so there's no added latency?

**A:**
- **Serilog** — zero blocking cost. Buffers internally, writes async. `LogContext.PushProperty` is in-memory only.
- **Phase 6 AuditLog** — small intentional cost. Writes in the same DB transaction as the business write. If the write rolls back, the audit row rolls back too. That consistency is worth the microsecond overhead.
- **True async separation** — already on the roadmap as Phase 9 (Outbox Pattern). Notifications, emails, downstream events write to an Outbox table; a background worker processes them. The request completes instantly.

### Should auditing be a toggleable module per customer?

**Q:** If a customer doesn't want auditing, should it be easy to turn off? Should features be sold as separate modules?

**A:** Yes, but the mechanism is feature flags / tenant configuration, not separate services. MediatR pipeline behaviours (`AuditBehaviour`, `LoggingBehaviour`) are already modular — they can be conditionally skipped per tenant config. A full "sell modules separately" entitlement system is a Phase 14+ commercial concern. Going microservices to solve this now would be massive over-engineering. Clean Architecture already gives you the extraction seams — use them when you actually need to, not speculatively.

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

### State Machine (Phase 2)
- `Incident.TransitionTo(newStatus)` enforces all status rules at the domain level — never in the handler or controller.
- `InvalidStatusTransitionException` → 422 Unprocessable Entity via `ExceptionHandlingMiddleware`.
- `AssignIncidentHandler` auto-transitions to `UnderInvestigation` when status is `Reported` — business rule lives in the handler, not the domain, because it involves coordination not invariants.

### PaginatedList (Phase 2)
- Two DB trips: `CountAsync` for total count, then `Skip/Take/Select` for the page. Do not load all rows.
- Filters are composed with nullable checks: `if (request.Status.HasValue) query = query.Where(...)`. Never filter on null.
- `PaginatedList<T>` lives in `Application/Common/Models/` — reusable across all features.

### Record with { } expression (Phase 2)
- Controllers use `command with { Id = id }` to merge route ID into the body command without manual mapping.
- Example: `await _sender.Send(command with { Id = id }, cancellationToken);`

### CorrectiveAction State Machine (Phase 3)
- `CorrectiveAction.TransitionTo()` follows same pattern as `Incident.TransitionTo()` — domain method, switch expression, guard clauses.
- Auto-stamps `CompletedAt` when → Completed; clears it when pushed back → InProgress.
- `Verified` and `Cancelled` are terminal — no transitions out. Enforced by the switch (no matching arm = throws).
- `UpdateCorrectiveActionCommandHandler` does NOT guard terminal states yet — tracked as EHS-37.

### Testing (from Phase 3 onwards)
- Tests ship with the code in the same commit — never deferred to a separate phase.
- Backend: xUnit + Moq + FluentAssertions.
- Frontend (Phase 12): React Testing Library.
- E2E (Phase 16): Playwright in GitHub Actions pipeline.

---

## What's Coming (Don't Build Early)

| Phase | What | When |
|---|---|---|
| Phase 2 | Incident status machine, pagination, filtering | ✅ Done |
| Phase 3 | Corrective Actions | ✅ Done |
| Phase 4 | Users & Authentication | Next |
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

## Auth & JWT — Full Architecture (Phase 4)

### Why ICurrentUserService exists

Every handler that does something protected needs to know: who is making this request? Their user ID, their tenant, their role. Without this service, every handler would reach directly into `HttpContext` — that is Infrastructure code leaking into Application, breaking Clean Architecture.

The **interface** lives in Application (the contract). The **implementation** lives in Infrastructure (reads from HttpContext). Handlers inject the interface and never know about HTTP. If you switch to background jobs, CLI, or a fake for tests — handlers don't change.

### The full JWT request cycle

```
LOGIN:
  POST /api/auth/login (email + password)
  → LoginCommandHandler reads User from DB
  → BCrypt.Verify(password, user.PasswordHash) → true
  → Build JWT claims: uid, tid, ClaimTypes.Role, ctype
  → Sign with our secret key
  → Return JWT string to client

EVERY SUBSEQUENT REQUEST:
  Client sends: Authorization: Bearer <jwt>
  → JWT middleware validates signature + expiry
  → Decodes payload → builds ClaimsPrincipal
  → Assigns to HttpContext.User automatically
  → CurrentUserService reads: HttpContext.User.FindFirstValue("uid")
  → Handler uses: _currentUserService.UserId
```

### Our JWT claim names

| Claim key | Contains | Example |
|---|---|---|
| `"uid"` | User.Id (our DB primary key) | `Guid` |
| `"tid"` | User.TenantId (Organisation they belong to) | `Guid` |
| `ClaimTypes.Role` | User.Role enum value | `"SafetyOfficer"` |
| `"ctype"` | User.CompanyType enum value | `"Client"` |

We define these names — they are not set by Microsoft or Okta. They are embedded in the JWT by our `LoginCommandHandler` at login time and read back by `CurrentUserService` on every request.

### BCrypt — why there is no PasswordSalt column

BCrypt embeds the salt inside the hash string itself. The output of `BCrypt.HashPassword("password")` is a single string like `"$2a$11$<salt+hash combined>"`. Verification is `BCrypt.Verify("password", storedHash)` — BCrypt extracts the salt internally. A separate `PasswordSalt` column is an artifact of older manual SHA-256 hashing and is not needed.

### JWT middleware setup (EHS-41)

```csharp
// Program.cs — what we add in EHS-41
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["Jwt:Secret"]!)),
            ValidateIssuer = false,
            ValidateAudience = false
        };
    });

app.UseAuthentication(); // decodes JWT → HttpContext.User
app.UseAuthorization();  // enforces [Authorize] attributes
```

### What changes when Azure AD / Okta arrives (Phase 14)

Their token is used **once at login** as proof of identity. After that, we issue our own JWT with our own claim names. `CurrentUserService` never changes — it always reads from our JWT.

```
Phase 4:  User authenticates with email + password → we issue our JWT
Phase 14: User authenticates with Microsoft/Okta → we validate their token ONCE
          → look up our DB → issue our JWT with same claim names → same CurrentUserService
```

### Refresh tokens — Phase 14 deferred decision

Phase 4 uses simple re-login: when our JWT expires (12 hours), client gets a 401 and re-authenticates. This is acceptable for early phases.

Phase 14 adds refresh tokens:
- **Access token** — short-lived (15-60 min), sent on every API call
- **Refresh token** — long-lived (7-30 days), stored securely, sent only to `POST /api/auth/refresh`
- When access token expires: client sends refresh token → we issue new access token → user never sees a logout
- If refresh token is stolen: revoke it in DB → forces re-login on next use

The user experience: logs in once, stays logged in for weeks, with access tokens rotating silently every hour.

---

## Future Phase Design Decisions (Don't Build Early)

### GDPR & Compliance — Phase 14+ concern
The platform has not been formally designed for GDPR, OSHA, or ISO 45001 compliance yet. Key conflicts to resolve:
- **Soft deletes vs Right to Erasure** — resolved via pseudonymisation (replace personal identifiers with anonymous tokens, keep audit records intact)
- **Data minimisation** — only collect what's needed (currently fine)
- **Data portability** — export a user's data on request (needs a future endpoint)
- **Breach notification** — 72-hour window requirement
- **Admin data access** — admins reading other users' data (even read-only "shadow mode") is a GDPR violation in many jurisdictions. Do not build impersonation or shadow mode without legal review.

Add a formal GDPR/compliance phase before the platform goes near real clients.

### Localisation / i18n — Phase 12+ concern
All hardcoded error messages (e.g. "Invalid email or password.") need to go through a resource file or message catalogue when localisation is added. Flag every hardcoded user-facing string for this phase.

### Resource-level Authorization — Phase 5/6 concern
Users who are named on a record (author, approver, assignee) must retain permanent read access regardless of role changes. Authorization check:
```
canView = hasRolePermission OR isAuthor OR isApprover OR isAssignee
```
Design this properly when building the audit log (Phase 6).

### Role Change & Audit Trail
When a user's role changes, historical records they created/approved must remain attributed to them with their name permanently. Role changes do not rewrite history. Implement via immutable `ChangedById` on AuditLog — the name is resolved from User at read time, not stored inline.

### Permissions Layer (Phase 8+)
One role per user is correct for Phase 4. When needed:
- Permissions layer sits on top of roles
- A role is a preset bundle of permissions
- Admins can add/remove individual permissions per user (overrides)
- Org admins can customise what each role can do in their org (configurable per company)
- Do NOT add multiple roles per user — use permission overrides instead

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
| Session 6 | Apr 2026 | Phase 2 DONE. EHS-21 through EHS-25 complete. State machine (TransitionTo), PaginatedList, UpdateIncident/UpdateStatus/AssignIncident commands, full controller with 6 endpoints. All tested via Swagger — 422 on invalid transitions confirmed working. Testing strategy locked: tests ship with code from Phase 3 onwards. |
| Session 7 | May 2026 | Phase 3 DONE (from previous session). Phase 4 started. EHS-32/37/38 domain encapsulation debt paid off. EHS-39 User entity + migration done. EHS-40 ICurrentUserService in progress. Auth/JWT full cycle documented. Technical debt EHS-44 (error messages) logged and deferred. Collaboration rule locked: code changes are Parth's, doc changes are Claude's. |
| Session 8 | May 2026 | EHS-41 DONE. AuthController + Register/Login commands + JWT generation + BCrypt hashing + 27 passing tests. Post-commit fixes: IsActive check on login, global email uniqueness enforced at DB level. EHSPlatform.Application.Tests project created. Deep design discussions: role-change audit trail, resource-level authorization, impersonation/shadow mode risks, GDPR compliance gap identified (needs future phase). |
| Session 9 | May 2026 | EHS-42 DONE. [Authorize] on IncidentsController + CorrectiveActionsController. [AllowAnonymous] on AuthController. ForbiddenAccessException → 403 in middleware. ReportedById wired from ICurrentUserService (removed from request body). Closed transition guarded to SafetyOfficer/OrganizationAdmin only. AssignIncidentHandler validates assignee exists and is active. 15 Application tests + 22 Domain tests all green. EHS-45 created: Architecture Spike — Concurrency, Consistency, Auditability and Traceability Strategy (identified during session as system-wide gap). |
| Session 10 | May 2026 | EHS-45 DONE. ADR-012 written and then expanded after multi-AI review (ChatGPT, Gemini, Claude, LLM Council). Final decisions: RowVersion + ETag/If-Match (both header and body), three audit tables (Entity + Security + BusinessRule) + Azure SQL Ledger Tables, FusionCache in-memory first then Redis as L2, scale threshold corrected to ~250 tenants, Elasticsearch adopted progressively (local Docker for learning, SQL Full-Text in production until Phase 11), rate limiting and idempotency deferred as tech debt. Redis and Elasticsearch both finalized — local Docker/Podman free forever, self-hosted VM for early production. Audit table structure (shared 3 vs per-entity 3) left OPEN for Phase 6 decision. EHS-48 through EHS-53 created. Sprint 3 closed, Sprint 4 started for Phase 5. |
| Session 11 | May 2026 | Carried-over fixes done: EHS-35 (FK constraints on Incidents + re-open from Closed role guard + 7 new tests), EHS-36 (PATCH /assign returns 200 with status body instead of 204), EHS-44 (DomainValidationException → 422, entity-agnostic InvalidStatusTransitionException, ToDisplayName() enum extensions, plain-language NotFoundException without raw GUIDs, UpdateCorrectiveAction terminal-state guard fixed from 500 → 422). 22 tests all green. Phase 5 next. |
| Session 12 | May 2026 | EHS-46 DONE. RowVersion optimistic concurrency: added byte[] RowVersion to BaseEntity, .IsRowVersion() on all 6 EF configs (Incident, CorrectiveAction, User, Organization, Site, SiteArea), DbUpdateConcurrencyException → 409 in ExceptionHandlingMiddleware, AddRowVersion migration. Migration FK conflict resolved by dropping orphaned dev data and rebuilding DB. 45 tests all green (23 application, 22 domain). |
| Session 13 | May 2026 | EHS-47 DONE. CorrelationId wired through Serilog enricher: CorrelationIdMiddleware (generates/reads X-Correlation-Id header, stores in HttpContext.Items, pushes to Serilog LogContext), IRequestContext interface in Application, RequestContext implementation in Infrastructure, Enrich.FromLogContext() + output template in Program.cs. CorrelationId verified in console output — same ID on all log lines per request. .claude/ added to .gitignore. Design Q&A documented: CorrelationId vs session tracing, async logging cost, modular tenant feature flags. |
| Session 14 | May 2026 | EHS-48 DONE. TenantId (Guid, required, non-nullable) added to Incident, CorrectiveAction, Site, SiteArea — all implement ITenantEntity. EF configs updated with IsRequired() + FK to Organization (Restrict) for each. AddTenantIdToEntities migration applied. 45 tests all green. Side session: AI-first strategy formalised — ai-first-strategy.md, ai-capabilities-research.md, semantic-form-engine-design.md all written and pushed to ehs-platform-docs. Semantic form engine design (three-layer: AiDescription metadata + IFormSemanticContextBuilder + ODK-style entity mapping) solves the AI-readability gap for Phase 7 dynamic forms — novel in EHS SaaS space. |
| Session 15 | May 2026 | EHS-49 DONE. ClientContractor entity added to Domain (BaseEntity + ITenantEntity, Name/ContractorCode/IsActive/IsDeleted), EF config with unique index on (TenantId, ContractorCode) + FK to Organizations (Restrict) + RowVersion. AddClientContractor migration applied. Data model only — no API endpoints, no CQRS. EHS-50 DONE. Pre-step: IsDeleted added to Site, SiteArea, Organization, User (all previously missing — architecture rule violation caught during audit). EF configs updated, AddIsDeletedToSiteSiteAreaUserOrganization migration applied. Global Query Filters: combined TenantId + IsDeleted on all 5 ITenantEntity types (Incident, CorrectiveAction, Site, SiteArea, ClientContractor); IsDeleted-only on User and Organization (no TenantId filter — login is global, Org IS the tenant). HasQueryFilter removed from IncidentConfiguration and CorrectiveActionConfiguration (DbContext owns all filters). TenantResolutionMiddleware added as lightweight guard — rejects authenticated requests missing tid claim. 46 tests all green including new TenantIsolationTests verifying TenantA data is invisible to TenantB. |

---

*Update this file at the end of every session. Commit it with the code.*
