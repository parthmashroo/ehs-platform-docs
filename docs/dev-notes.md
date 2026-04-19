# EHS Platform — Dev Notes

## Phase 1 Complete — Core Domain + Incident Reporting

**Completed:** April 2026
**Stories:** EHS-11 through EHS-18

---

## What Was Built

### Domain Layer (EHSPlatform.Domain)
- `BaseEntity.cs` — Id (Guid), CreatedAt, UpdatedAt
- `ITenantEntity.cs` — TenantId interface for all tenant-scoped entities
- Enums: `IncidentStatus`, `IncidentType`, `Severity`, `CompanyType`, `CorrectiveActionStatus`, `UserRole`
- Entities: `Organization`, `Site`, `SiteArea`, `Incident`

### Infrastructure Layer (EHSPlatform.Infrastructure)
- `ApplicationDbContext` with DbSets for all entities
- `IEntityTypeConfiguration` per entity in `Configurations/` folder
- `IApplicationDbContext` interface in Application layer
- EF Core migration: `InitialCreate` — creates Organizations, Sites, SiteAreas, Incidents tables
- `DependencyInjection.cs` wires up EF Core

### Application Layer (EHSPlatform.Application)
- MediatR 12.2.0 + FluentValidation 11.9.0
- `CreateIncidentCommand` + Handler + Validator
- `GetIncidentsQuery` + Handler → returns `List<IncidentDto>`
- `GetIncidentByIdQuery` + Handler → returns single `IncidentDto` or throws `NotFoundException`
- `ValidationBehaviour` in MediatR pipeline — auto-validates every command
- `DependencyInjection.cs` registers MediatR and FluentValidation

### API Layer (EHSPlatform.API)
- `IncidentsController` — POST, GET /api/incidents, GET /api/incidents/{id:guid}
- `ISender` injected (not IMediator — leaner interface)
- `CreatedAtAction` on POST returns 201 with Location header
- `{id:guid}` route constraint — auto-rejects non-Guid ids with 400

---

## How to Run Locally

### Prerequisites
- .NET 8 SDK
- SQL Server 2022 (local)
- Connection string: `Server=localhost;Database=EHSPlatform;Trusted_Connection=True;TrustServerCertificate=True;`

### Run
```powershell
dotnet run --project src/EHSPlatform.API
```

Then open: `http://localhost:5147/swagger`

### Endpoints in Swagger
- `POST /api/incidents` — create a new incident
- `GET /api/incidents` — list all incidents
- `GET /api/incidents/{id}` — get incident by Guid

---

## Key Decisions Made in Phase 1

- **No auth yet** — TenantId and UserId are hardcoded placeholders. Auth comes in Phase 4.
- **ISender over IMediator** — controllers only need Send(), not the full IMediator interface
- **NotFoundException** — thrown by GetIncidentByIdQuery handler when incident not found. ExceptionHandlingMiddleware converts it to 404 Problem Details automatically.
- **IncidentDto** — separate read model returned by queries. Never expose the domain entity directly from the API.

---

## What's Next — Phase 2: Incident Lifecycle (Status Machine)

- `Incident.TransitionTo(newStatus)` domain method enforcing state machine rules
- `InvalidStatusTransitionException`
- `UpdateIncidentCommand` + `UpdateIncidentStatusCommand` + `AssignIncidentCommand`
- `PaginatedList<T>` — paginated GET /api/incidents with filters (status, severity, type, site)
