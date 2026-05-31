# EHS Platform — Phase 1–5 Ticket History

> Archive. Read only when tracing a specific past ticket or decision.

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

## Phase 2 Ticket Status

| Ticket | Description | Status |
|---|---|---|
| EHS-21 | Incident.TransitionTo() domain method + InvalidStatusTransitionException | ✅ Done |
| EHS-22 | UpdateIncidentCommand, UpdateIncidentStatusCommand, AssignIncidentCommand + handlers + validators | ✅ Done |
| EHS-23 | PaginatedList\<T\>, GetIncidentsQuery with pagination and filters | ✅ Done |
| EHS-24 | PUT, PATCH /status, PATCH /assign endpoints in IncidentsController | ✅ Done |
| EHS-25 | Docs updated for Phase 2 completion | ✅ Done |

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

## Phase 5 Ticket Status

| Ticket | Description | Status |
|---|---|---|
| EHS-46 | RowVersion on BaseEntity + .IsRowVersion() on all 6 EF configs + DbUpdateConcurrencyException → 409 + migration + concurrency test | ✅ Done |
| EHS-47 | CorrelationId: Serilog enricher + IRequestContext + stamp on all audit rows | ✅ Done |
| EHS-48 | TenantId on all existing entities + combined migration (with EHS-46 RowVersion) | ✅ Done |
| EHS-49 | ClientContractor entity + EF config + migration | ✅ Done |
| EHS-50 | EF Core Global Query Filters + TenantResolutionMiddleware | ✅ Done |
| EHS-51 | Redis `IDistributedCache` + `ICacheService` abstraction + Infrastructure tests | ✅ Done |
| EHS-52 | Elasticsearch + Kibana local Docker/Podman setup + SQL Full-Text comparison spike | ✅ Done |
| EHS-53 | Phase 5 docs update | ✅ Done |
