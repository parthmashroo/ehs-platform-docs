# EHS Platform — Code Patterns Reference

> Read when adding a new entity, writing a command, or checking a pattern.
> Not needed at session start.

---

## New Entity Checklist — Follow This Every Time

### Step 1 — Domain: Enums
`EHSPlatform.Domain/Enums/` — one file per enum, explicit integer values starting at 1.

```csharp
public enum MyStatusEnum { Active = 1, Inactive = 2, Archived = 3 }
```

### Step 2 — Domain: Entity
`EHSPlatform.Domain/Entities/MyEntity.cs` — extend `BaseEntity`, string defaults to `string.Empty`, booleans default explicitly.

```csharp
public class MyEntity : BaseEntity, ITenantEntity
{
    public Guid TenantId { get; set; }
    public string Title { get; set; } = string.Empty;
    public MyStatusEnum Status { get; set; } = MyStatusEnum.Active;
    public bool IsDeleted { get; set; } = false;
    public Guid? ParentEntityId { get; set; }
}
```

### Step 3 — Infrastructure: EF Configuration
`EHSPlatform.Infrastructure/Persistence/Configurations/MyEntityConfiguration.cs`

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
        builder.Property(x => x.RowVersion).IsRowVersion();

        builder.HasOne<ParentEntity>()
            .WithMany()
            .HasForeignKey(x => x.ParentEntityId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.HasIndex(x => x.ParentEntityId)
            .HasFilter("[ParentEntityId] IS NOT NULL");
    }
}
```

**Note:** Global Query Filters (TenantId + IsDeleted) are set in `ApplicationDbContext.OnModelCreating()` — do NOT add `HasQueryFilter` in the configuration file.

### Step 4 — Infrastructure: Register DbSet
Update both files together — if you update one and forget the other, it will not compile.

**`ApplicationDbContext.cs`:**
```csharp
public DbSet<MyEntity> MyEntities => Set<MyEntity>();
```

**`IApplicationDbContext.cs`:**
```csharp
DbSet<MyEntity> MyEntities { get; }
```

### Step 5 — Infrastructure: Migration
```bash
dotnet ef migrations add AddMyEntity --project src/EHSPlatform.Infrastructure --startup-project src/EHSPlatform.API
dotnet ef database update --project src/EHSPlatform.Infrastructure --startup-project src/EHSPlatform.API
```
Review the generated migration before running update. Check: correct table name, correct column types, FK constraint, filtered index.

### Step 6 — Application: DTO
`EHSPlatform.Application/MyEntities/Queries/MyEntityDto.cs` — never return raw entities from handlers.

```csharp
public class MyEntityDto
{
    public Guid Id { get; init; }
    public string Title { get; init; } = string.Empty;
    public string Status { get; init; } = string.Empty;
    public DateTimeOffset CreatedAt { get; init; }
}
```

### Step 7 — Application: Commands (Writes)
Three files per command — always. `Commands/CreateMyEntity/`

```csharp
// Command
public record CreateMyEntityCommand : IRequest<Guid>
{
    public string Title { get; init; } = string.Empty;
}

// Handler
public class CreateMyEntityCommandHandler : IRequestHandler<CreateMyEntityCommand, Guid>
{
    private readonly IApplicationDbContext _context;
    public CreateMyEntityCommandHandler(IApplicationDbContext context) => _context = context;

    public async Task<Guid> Handle(CreateMyEntityCommand request, CancellationToken ct)
    {
        var entity = new MyEntity
        {
            Title = request.Title,
            CreatedAt = DateTimeOffset.UtcNow,
            UpdatedAt = DateTimeOffset.UtcNow
        };
        _context.MyEntities.Add(entity);
        await _context.SaveChangesAsync(ct);
        return entity.Id;
    }
}

// Validator
public class CreateMyEntityCommandValidator : AbstractValidator<CreateMyEntityCommand>
{
    public CreateMyEntityCommandValidator()
    {
        RuleFor(x => x.Title).NotEmpty().MaximumLength(500);
    }
}
```

### Step 8 — Application: Queries (Reads)
```csharp
public async Task<PaginatedList<MyEntityDto>> Handle(GetMyEntitiesQuery request, CancellationToken ct)
{
    var safePageSize = Math.Min(request.PageSize, PaginationConstants.MaxPageSize);
    var query = _context.MyEntities.AsQueryable();
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

### Step 9 — API: Controller
```csharp
[ApiController]
[Route("api/[controller]")]
public class MyEntitiesController : ControllerBase
{
    private readonly ISender _sender;
    public MyEntitiesController(ISender sender) => _sender = sender;

    [HttpPost]
    [EndpointSummary("Plain language description for end users")]
    public async Task<IActionResult> Create(CreateMyEntityCommand command, CancellationToken ct)
    {
        var id = await _sender.Send(command, ct);
        return CreatedAtAction(nameof(GetById), new { id }, new { id });
    }
}
```
Rules: `ISender` not `IMediator`. POST returns 201 `CreatedAtAction`. PUT/PATCH returns 204. Zero business logic in controllers.

### Step 10 — Tests
Minimum per entity:
- Unit test: any domain method with rules (TransitionTo, AssignTo) — test every valid and invalid path
- Integration test: happy path for Create + Get, and one failure case (not found, invalid input)

Tests ship in the same commit as the code. Never deferred.

### Step 11 — Docs & Jira
1. Update `START_HERE.md` — session handoff block
2. Update `docs/current/phase-X-status.md` — ticket status
3. Commit docs to `ehs-platform-docs`
4. Transition Jira ticket → Done

---

## MediatR Pipeline Behaviours (order they run)

```
1. ValidationBehaviour    → FluentValidation on every command
2. LoggingBehaviour       → logs command name + execution time
3. AuditBehaviour         → records writes to AuditLog table (Phase 6)
→ Handler                 → actual business logic
```

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
- Any state → Reported:  Re-opened — creates audit entry

InvalidStatusTransitionException thrown for any invalid jump.
```

---

## Key Domain Enums

| Enum | Values |
|---|---|
| `CompanyType` | Client = 1, Contractor = 2 |
| `IncidentType` | Injury, NearMiss, HazardousCondition, EquipmentFailure, EnvironmentalHazard, SafetyViolation, ChemicalSpill, Other |
| `Severity` | Low, Medium, High, Critical |
| `IncidentStatus` | Reported, UnderInvestigation, AwaitingAction, Resolved, Closed |
| `CorrectiveActionStatus` | Open, InProgress, Completed, Verified, Cancelled |
| `Priority` | Low = 1, Medium = 2, High = 3, Critical = 4 |
| `UserRole` | OrganizationAdmin, SafetyOfficer, ComplianceManager, IncidentReviewer, SiteEmployee, ContractorAdmin, ContractorSupervisor, ContractorWorker |
| `AuditAction` | Created, Updated, Deleted |

---

## Lessons Learned / Gotchas

### EF Core
- Global Query Filters must be set in `ApplicationDbContext.OnModelCreating()` — NOT in `IEntityTypeConfiguration` files. Setting in both causes double filters.
- `IsDeleted` was missing from `Site`, `SiteArea`, `Organization`, `User` when Phase 5 started — always add `IsActive` + `IsDeleted` to every new entity.
- Use `.Select()` to project to DTOs directly in queries — never return raw entities from handlers.
- Always use a stable sort: `.OrderByDescending(x => x.CreatedAt).ThenBy(x => x.Id)`
- Always cap pageSize: `Math.Min(request.PageSize, PaginationConstants.MaxPageSize)`

### MediatR
- Use `ISender` not `IMediator` in controllers — cleaner, only exposes `Send`.
- `record` types for Commands and Queries — immutable by default.
- Pass `CancellationToken` through to `_sender.Send()`.

### DateTimeOffset (ADR-016)
- All timestamp fields: `DateTimeOffset`. All assignments: `DateTimeOffset.UtcNow`.
- Only exception: `JwtTokenService.cs` — `SecurityTokenDescriptor.Expires` requires `DateTime?`.
- InMemory provider silently ignores `HasColumnType("datetimeoffset(7)")` — column type tests require real SQL Server (Testcontainers).

### State Machine
- `Incident.TransitionTo(newStatus)` enforces all status rules at the domain level — never in the handler.
- `AssignIncidentHandler` auto-transitions to `UnderInvestigation` when status is `Reported` — coordination logic in the handler, invariant logic in the domain.
- CorrectiveAction: auto-stamps `CompletedAt` when → Completed; clears it when pushed back → InProgress.

### Controllers
- `command with { Id = id }` to merge route ID into body command without manual mapping.
- `{id:guid}` route constraint — auto-rejects non-Guid ids with 400 before hitting the handler.
