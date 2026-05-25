# EHS Platform — Full Technical Roadmap & Master Reference



## About This Document

This is the single source of truth for building the EHS (Environment, Health & Safety) SaaS platform.

Written so any AI assistant can resume this project with full context from any session.



**Developer:** .NET backend engineer, 11+ years experience. Learning modern stack while building.

**Goal:** Build a real production-quality EHS SaaS system — not a tutorial app.

**Practice machine:** Personal machine only. Never on company laptop.

**Current status:** Phase 0 ✅ Phase 1 ✅ Phase 2 ✅ Phase 3 ✅ Phase 4 ✅ Phase 5 ✅ Phase 6 next.



---



## What We Are Building



A **full EHS (Environment, Health & Safety) SaaS platform** for safety-critical industries

(construction, manufacturing, chemical plants, mining, energy).



Inspired by a real enterprise system (MAISAAS) the developer worked on previously.

That system was built on .NET Core 3.1, layered architecture, Razor MVC.

We are building the modern equivalent — better architecture, better stack, cloud-native.



### Two Types of Companies Use This Platform



**Client (Site Owner):**

The company that owns the workplace/site. A factory, construction site, oil platform.

They have authority, oversight, compliance responsibility, and approval power.

They manage safety across their site, including contractors who come to work there.



**Contractor:**

The company that sends workers to perform work at the client's site.

They execute work, submit compliance documents, report incidents, complete training.



This Client vs Contractor distinction is fundamental. It drives the entire authorization model,

the data model (CompanyType enum), and what each user can see and do.



---



## The Anti-Procrastination Rules (Non-Negotiable)



The developer is a self-identified procrastinator and overthinker. These rules prevent abandonment:



1. **One phase at a time.** Never plan Phase 5 while building Phase 1.

2. **Each session ends with working, committed code.** No half-finished features.

3. **No gold-plating early phases.** Phase 1 is intentionally ugly. That is correct.

4. **Decisions are made once.** Technology choices are locked. Don't revisit.

5. **Done is better than perfect.** A working ugly endpoint beats a perfectly designed nothing.

6. **Minimum viable phase.** If a phase feels too big, cut it in half. Ship something.

7. **No rabbit holes.** New interesting ideas go in a future phase note. Not now.



**If the developer starts scope-creeping or re-debating decided things, remind them of these rules.**



---



## Decisions Already Made (Never Revisit)



### Portal Architecture

- **Single React app with role-based views.** NOT two separate apps.

- One codebase, one deployment. The app reads the user's role/company type on login and renders

  different dashboards, navigation, and screens accordingly.

- Later if needed: `client.app.com` and `contractor.app.com` subdomains via DNS + nginx config

  pointing at the same deployed app. Zero extra code. Subdomain context sets initial routing.



### Form Engine Scope

- Core business entities (Incident, CorrectiveAction, WorkPermit, etc.) stay as **proper typed entities**

  with strongly typed fields in the database.

- The Dynamic Form Engine handles **supplementary data collection** — risk assessment checklists,

  site inspection forms, toolbox talk records, custom questionnaires, audit checklists, induction forms.

- How/whether to integrate core module forms with the form engine is a future decision.



### Form Engine & Templates

- Phase 7 ships 8 pre-built industry/regulatory templates with full `AiDescription` pre-populated.
  These are `IsSystemTemplate = true` — read-only, tenants clone and customise.
- Standards covered in Phase 7 starter pack: ISO 45001, OSHA 1904, CDM 2015, NFPA 51B,
  OSHA 1910.146, OSHA 1910.147 (see ADR-013 and `semantic-form-engine-design.md`).
- Additional regulatory packs (ISO 9001, ISO 14001, RIDDOR, sector-specific) are a commercial
  product decision — built when paying customers request them.



### Notifications

- **Outbox Pattern** — when a command executes, it saves a domain event to an OutboxMessage table

  in the same database transaction. A background worker reads and processes it (sends email).

- No RabbitMQ for now. Azure Service Bus added in a later phase when scale requires it.



### Auth

- JWT generated **inside the API** — no separate IdentityServer or auth service.

- SCIM 2.0 / Azure AD / Okta SSO integration added in a later phase for enterprise customers.



---



## Full Technology Stack



### Backend

| Concern | Technology | Why |

|---|---|---|

| Framework | .NET 8 Web API | Modern LTS |

| Architecture | Clean Architecture | Domain / Application / Infrastructure / API layers |

| Pattern | CQRS + MediatR | Separates reads/writes, pipeline behaviors |

| ORM | Entity Framework Core 8 | Code-first, migrations, interceptors for audit |

| Validation | FluentValidation | Declarative, in MediatR pipeline |

| Mapping | Mapster | Faster than AutoMapper |

| Logging | Serilog | Structured logging, Azure App Insights sink |

| Auth | JWT Bearer tokens | Standard, stateless |

| Caching | Redis (StackExchange.Redis) | Distributed cache |

| Resilience | Polly | Retry, circuit breaker |

| Background Jobs | BackgroundService + PeriodicTimer | Built-in .NET 8 |

| Notifications | Outbox Pattern → email | Reliable async delivery |

| Testing | xUnit + Moq + FluentAssertions | Industry standard |

| API Docs | Swagger / Scalar | Auto-generated |

| Error Format | Problem Details (RFC 7807) | Standard HTTP errors |



### Frontend

| Concern | Technology |

|---|---|

| Framework | React 18 + TypeScript |

| Build Tool | Vite |

| Routing | React Router v6 |

| Server State | TanStack Query (React Query) |

| Client State | Zustand |

| UI Components | shadcn/ui + Tailwind CSS |

| HTTP Client | Axios (with auth interceptors) |

| Forms | React Hook Form + Zod |



### Infrastructure

| Concern | Technology |

|---|---|

| Containerization | Docker + Docker Compose |

| Hosting | Azure Container Apps |

| Database | Azure SQL (SQL Server locally via Docker) |

| Caching | Azure Cache for Redis |

| Secrets | Azure Key Vault |

| File Storage | Azure Blob Storage |

| Messaging | Azure Service Bus (Phase 15+) |

| Monitoring | Azure Application Insights |

| CI/CD | GitHub Actions |



### Architecture Decisions (Locked)

- **Multi-tenancy:** Row-level isolation via EF Core Global Query Filters on TenantId.

  Single database, all tenants share tables, filtered automatically.

- **CompanyType:** Every Organization is either `Client` (1) or `Contractor` (2). Fundamental.

- **Site Hierarchy:** Organization → Site → SiteArea. Incidents, permits, etc. link to a Site.

- **Audit logging:** EF Core SaveChanges interceptor captures before/after on all auditable entities.

- **CQRS:** Commands (writes) and Queries (reads) are separate MediatR handlers. No sharing.

- **Soft deletes:** IsActive / IsDeleted flag everywhere. No hard deletes.

- **Permission system:** Module-level + Action-level permissions per role (not just role names).

- **Lookup tables:** Some domain data (e.g., SiteArea names, custom incident categories) is DB-driven

  per tenant, not hardcoded enums. This allows tenant customisation without code changes.

- **Outbox pattern:** Domain events persisted to OutboxMessage in same transaction as business data.

  Background worker processes and delivers. Replaces need for message broker in early phases.



### Clean Architecture Layers

- `Domain` — Entities, Value Objects, Enums, Domain Events, Interfaces. Zero dependencies.

- `Application` — Commands, Queries, Handlers, DTOs, Validators, Interfaces. Depends on Domain only.

- `Infrastructure` — EF Core, Repositories, Azure clients, Email, Cache. Depends on Application.

- `API` — Controllers, Middleware, Startup. Depends on Application (not Infrastructure directly).



### MediatR Pipeline (every request goes through this)

```

ValidationBehaviour     → auto-validates every command via FluentValidation

LoggingBehaviour        → logs every command/query with execution time

AuditBehaviour          → records every write to AuditLog table

→ Command/Query Handler → business logic

```



### Middleware Pipeline Order

```

1. CorrelationIdMiddleware      — assigns request ID

2. ExceptionHandlingMiddleware  — catches all errors, returns Problem Details

3. Serilog request logging

4. HTTPS redirection

5. Authentication               — validates JWT

6. Authorization                — checks roles/policies

7. TenantResolutionMiddleware   — extracts TenantId + CompanyType from JWT

8. Controllers

```



---



## User Roles (Both Portals, Single App)



### Client-Side Roles (Site Owner / Safety Management Portal)

| Role | Responsibilities |

|---|---|

| **Organization Admin** | Full setup: sites, areas, users, configuration, document types, workflows |

| **Safety Officer** | Manage incidents, assign investigators, approve status transitions, close incidents |

| **Compliance Manager** | Conduct audits, verify corrective actions, manage compliance reports |

| **Incident Reviewer** | Review reported incidents, approve severity, authorize corrective actions |

| **Site Employee** | Report incidents, complete assigned corrective actions, complete training |



### Contractor-Side Roles (Contractor / Worker Portal)

| Role | Responsibilities |

|---|---|

| **Contractor Admin** | Manage contractor company, users, worker documents, site access requests |

| **Contractor Supervisor** | Manage team compliance, submit documents, track work permits |

| **Contractor Worker** | Report incidents on site, complete training, check in/out, fill forms |



### System-Level Role

| Role | Responsibilities |

|---|---|

| **Platform Admin (SuperAdmin)** | Manages the SaaS platform itself. Not a tenant user. |



---



## Data Model



### Organization (Tenant)

```

Organization

- Id (Guid)

- Name (string)

- CompanyType (enum: Client=1, Contractor=2)     ← CRITICAL distinction

- Industry (enum: Chemical, Manufacturing, Construction, Mining, Energy, Other)

- IsActive (bool)

- CreatedAt (DateTime)

```



### Site (Location owned by an Organization)

```

Site

- Id (Guid)

- TenantId (Guid) → Organization

- Name (string)

- Address (string?)

- City (string?)

- Country (string?)

- IsActive (bool)

- CreatedAt (DateTime)

```



### SiteArea (Zone/Area within a Site)

```

SiteArea

- Id (Guid)

- TenantId (Guid)

- SiteId (Guid) → Site

- Name (string)           — "Zone A", "Chemical Storage", "Loading Bay"

- IsActive (bool)

```



### ClientContractor (Which contractors work at which client sites)

```

ClientContractor

- Id (Guid)

- ClientOrganizationId (Guid) → Organization (must be Client type)

- ContractorOrganizationId (Guid) → Organization (must be Contractor type)

- IsActive (bool)

- ApprovedAt (DateTime?)

- CreatedAt (DateTime)

```



### User

```

User

- Id (Guid)

- TenantId (Guid) → Organization

- Email (string, globally unique — enforced at DB level, EHS-41)

- FullName (string)

- PasswordHash (string)    ← BCrypt embeds salt — no PasswordSalt column needed

- Role (enum: OrganizationAdmin, SafetyOfficer, ComplianceManager, IncidentReviewer,

              SiteEmployee, ContractorAdmin, ContractorSupervisor, ContractorWorker)

- IsActive (bool)

- LastLoginAt (DateTime?)

- CreatedAt (DateTime)

```



### Incident (Core Entity)

```

Incident

- Id (Guid)

- TenantId (Guid) → Organization

- SiteId (Guid) → Site                    ← links to site, not free text

- SiteAreaId (Guid?) → SiteArea

- Title (string)

- Description (string)

- Type (enum: Injury, NearMiss, HazardousCondition, EquipmentFailure,

              EnvironmentalHazard, SafetyViolation, ChemicalSpill, Other)

- Severity (enum: Low, Medium, High, Critical)

- Status (enum: Reported, UnderInvestigation, AwaitingAction, Resolved, Closed)

- OccurredAt (DateTime)

- ReportedById (Guid) → User

- AssignedToId (Guid?) → User             ← investigator

- RootCause (string?)

- IsDeleted (bool)                        ← soft delete

- CreatedAt (DateTime)

- UpdatedAt (DateTime)

```



### CorrectiveAction

```

CorrectiveAction

- Id (Guid)

- TenantId (Guid)

- IncidentId (Guid?) → Incident           ← nullable — CA can be standalone or from other source
- AuditFindingId (Guid?) → AuditFinding   ← Phase 15 (add when needed, same pattern)
                                            ← future: RiskId?, NonConformanceId? same pattern

- Title (string)

- Description (string)

- Status (enum: Open, InProgress, Completed, Verified, Cancelled)

- Priority (enum: Low, Medium, High, Critical)

- AssignedToId (Guid) → User

- DueDate (DateTime)

- CompletedAt (DateTime?)

- VerifiedById (Guid?) → User

- VerificationNote (string?)

- IsDeleted (bool)

- CreatedAt (DateTime)

- UpdatedAt (DateTime)

```

Source linking decision: ADR-011 — Separate nullable FKs per source type.
One CA has at most one source (or none). One source can have many CAs (one-to-many).
See ADR-011 for full reasoning including scale analysis at 350M rows.

### AuditLog (Immutable — every change recorded)

```

AuditLog

- Id (Guid)

- TenantId (Guid)

- EntityName (string) — "Incident", "CorrectiveAction"

- EntityId (string)

- Action (enum: Created, Updated, Deleted)

- ChangedById (Guid) → User

- ChangedAt (DateTime)

- OldValues (json string)

- NewValues (json string)

```



### OutboxMessage (Outbox Pattern for notifications)

```

OutboxMessage

- Id (Guid)

- TenantId (Guid)

- EventType (string) — "IncidentCreated", "CorrectiveActionOverdue"

- Payload (json string)

- CreatedAt (DateTime)

- ProcessedAt (DateTime?)

- Error (string?)

- RetryCount (int)

```



### IncidentAttachment (Phase 10+)

```

IncidentAttachment

- Id (Guid)

- TenantId (Guid)

- IncidentId (Guid)

- FileName (string)

- BlobUrl (string)

- UploadedById (Guid)

- UploadedAt (DateTime)

```



### Dynamic Form Engine (Phase 7)

```

FormTemplate

- Id (Guid)

- TenantId (Guid?) — null = system/ISO template (read-only for tenants)

- Name (string)

- Description (string)

- Category (enum: RiskAssessment, SiteInspection, ToolboxTalk, Induction, Audit,

ContractorOnboarding, Custom)

- IsSystemTemplate (bool)

- SourceTemplateId (Guid?) — if cloned from system template

- IsoStandard (string?) — "ISO 45001:2018", "OSHA", etc.

- Version (string) — "1.0"

- IsActive (bool)

- CreatedAt (DateTime)



FormSection

- Id (Guid)

- FormTemplateId (Guid)

- Title (string)

- Description (string?)

- Order (int)

- IsRepeatable (bool) — section can repeat (e.g. multiple witnesses)



FormField

- Id (Guid)

- FormSectionId (Guid)

- Label (string)

- FieldKey (string) — machine key: "witness_name", "severity_rating"

- FieldType (enum: Text, TextArea, Number, Date, DateTime, Checkbox, Radio,

Dropdown, MultiSelect, FileUpload, Signature, Rating, YesNo)

- IsRequired (bool)

- HelpText (string?)

- Placeholder (string?)

- Order (int)

- Options (json) — ["Low","Medium","High"] for dropdowns/radios

- ValidationRules (json) — {"min":0,"max":100,"maxLength":500}

- ConditionalLogic (json) — {"showIf":{"fieldKey":"severity","equals":"High"}}

- DefaultValue (string?)



FormSubmission

- Id (Guid)

- TenantId (Guid)

- FormTemplateId (Guid)

- FormTemplateVersion (string) — snapshot: which version was used

- LinkedEntityType (string?) — "Incident", "WorkPermit"

- LinkedEntityId (Guid?)

- SubmittedByUserId (Guid)

- SubmittedAt (DateTime)

- Status (enum: Draft, Submitted, UnderReview, Approved, Rejected)



FormFieldAnswer

- Id (Guid)

- FormSubmissionId (Guid)

- FormFieldId (Guid)

- FieldKey (string) — denormalized for query performance

- Value (string) — all types serialized to string/json

- DisplayValue (string?) — human-readable version

```



---



## Incident Status State Machine

```

Reported → UnderInvestigation → AwaitingAction → Resolved → Closed

↑ ↓

└──── (re-open) ───────┘



Rules:

- Reported: Initial state on creation

- UnderInvestigation: Requires AssignedToId to be set

- AwaitingAction: Requires RootCause to be filled

- Resolved: Requires all linked CorrectiveActions to be Completed or Verified

- Closed: Final state. Only SafetyOfficer or OrganizationAdmin

- Any state → Reported: "Re-opened" — adds audit entry

```



---



## Core API Endpoints



### Incidents

```

POST /api/incidents — Report new incident

GET /api/incidents — List (filterable, paginated)

GET /api/incidents/{id} — Get with full detail

PUT /api/incidents/{id} — Update details

PATCH /api/incidents/{id}/status — Transition status (state machine enforced)

PATCH /api/incidents/{id}/assign — Assign investigator

DELETE /api/incidents/{id} — Soft delete (Admin only)

GET /api/incidents/{id}/audit-log — Full change history

```



### Corrective Actions

```

POST /api/correctiveactions                      — log a new corrective action
GET /api/correctiveactions                       — list (filterable by incidentId, status, paginated)
GET /api/correctiveactions/{id}                  — get corrective action details
PUT /api/correctiveactions/{id}                  — update fields
PATCH /api/correctiveactions/{id}/status         — transition status (state machine enforced)
DELETE /api/correctiveactions/{id}               — soft delete

```



### Auth

```

POST /api/auth/register — Creates Organization + first Admin user

POST /api/auth/login — Returns JWT with userId, tenantId, companyType, email, role

POST /api/auth/refresh — Refresh token

```



### Users & Sites

```

GET /api/users — List users in tenant

POST /api/users/invite — Invite user

GET /api/sites — List sites for tenant

POST /api/sites — Create site

GET /api/sites/{id}/areas — List areas for site

```



### Form Engine (Phase 7+)

```

GET /api/form-templates — List templates (system + own)

POST /api/form-templates — Create template

GET /api/form-templates/{id} — Get template with sections and fields

PUT /api/form-templates/{id} — Update template

POST /api/form-templates/{id}/clone — Clone a system template

POST /api/form-submissions — Submit a form

GET /api/form-submissions/{id} — Get submission with answers

```



### Dashboard (Phase 11+)

```

GET /api/dashboard/summary — Open incidents, overdue CAs, by severity

GET /api/reports/incidents — Filtered export

```



---



## Solution Structure



```

EHSPlatform/

├── src/

│ ├── EHSPlatform.Domain/

│ │ ├── Entities/

│ │ │ ├── Incident.cs

│ │ │ ├── CorrectiveAction.cs

│ │ │ ├── Organization.cs

│ │ │ ├── Site.cs

│ │ │ ├── SiteArea.cs

│ │ │ ├── User.cs

│ │ │ ├── AuditLog.cs

│ │ │ ├── OutboxMessage.cs

│ │ │ ├── ClientContractor.cs

│ │ │ └── (FormTemplate, FormSection, FormField, FormSubmission, FormFieldAnswer — Phase 7)

│ │ ├── Enums/

│ │ │ ├── IncidentStatus.cs

│ │ │ ├── IncidentType.cs

│ │ │ ├── Severity.cs

│ │ │ ├── UserRole.cs

│ │ │ ├── CompanyType.cs ← Client vs Contractor

│ │ │ ├── CorrectiveActionStatus.cs

│ │ │ └── (FormFieldType.cs — Phase 7)

│ │ ├── Exceptions/

│ │ │ ├── DomainException.cs

│ │ │ ├── NotFoundException.cs

│ │ │ ├── ForbiddenException.cs

│ │ │ └── InvalidStatusTransitionException.cs

│ │ └── Common/

│ │ ├── BaseEntity.cs — Id, CreatedAt, UpdatedAt

│ │ └── ITenantEntity.cs — TenantId interface

│ │

│ ├── EHSPlatform.Application/

│ │ ├── Common/

│ │ │ ├── Interfaces/

│ │ │ │ ├── IApplicationDbContext.cs

│ │ │ │ ├── ICurrentUserService.cs

│ │ │ │ └── ICacheService.cs

│ │ │ ├── Behaviours/

│ │ │ │ ├── ValidationBehaviour.cs

│ │ │ │ ├── LoggingBehaviour.cs

│ │ │ │ └── AuditBehaviour.cs

│ │ │ └── Models/

│ │ │ ├── PaginatedList.cs

│ │ │ └── Result.cs

│ │ ├── Incidents/

│ │ │ ├── Commands/

│ │ │ │ ├── CreateIncident/

│ │ │ │ ├── UpdateIncident/

│ │ │ │ └── UpdateIncidentStatus/

│ │ │ └── Queries/

│ │ │ ├── GetIncidents/

│ │ │ └── GetIncidentById/

│ │ └── CorrectiveActions/

│ │ └── (same pattern)

│ │

│ ├── EHSPlatform.Infrastructure/

│ │ ├── Persistence/

│ │ │ ├── ApplicationDbContext.cs

│ │ │ ├── Configurations/ — IEntityTypeConfiguration per entity

│ │ │ ├── Migrations/

│ │ │ └── Interceptors/

│ │ │ └── AuditInterceptor.cs

│ │ ├── Identity/

│ │ │ └── JwtTokenService.cs

│ │ ├── Caching/

│ │ │ └── RedisCacheService.cs

│ │ ├── BackgroundJobs/

│ │ │ ├── OutboxProcessorJob.cs — processes OutboxMessage table

│ │ │ └── OverdueCaJob.cs

│ │ └── DependencyInjection.cs

│ │

│ └── EHSPlatform.API/

│ ├── Controllers/

│ │ ├── IncidentsController.cs

│ │ ├── CorrectiveActionsController.cs

│ │ ├── AuthController.cs

│ │ └── SitesController.cs

│ ├── Middleware/

│ │ ├── ExceptionHandlingMiddleware.cs

│ │ ├── TenantResolutionMiddleware.cs

│ │ └── CorrelationIdMiddleware.cs

│ └── Program.cs

│

├── tests/

│ ├── EHSPlatform.Domain.Tests/

│ ├── EHSPlatform.Application.Tests/

│ └── EHSPlatform.API.Tests/

│

├── frontend/ — React 18 + TypeScript (Phase 12)

│ └── (Vite, shadcn/ui, TanStack Query, Zustand, React Router)

│

├── docker-compose.yml — SQL Server + Redis

├── .github/workflows/ci-cd.yml

└── README.md

```



---



## Phases of Development



> Rule: **Each phase produces working, committed, runnable code.**

> No phase starts until the previous one is DONE.



---



### Phase 0: Project Skeleton ✅ COMPLETE

**Goal:** A running .NET 8 API that returns a health check. Nothing more.



- [ ] Create GitHub repository

- [ ] `dotnet new sln` — solution with 4 projects (Domain, Application, Infrastructure, API)

- [ ] Project references: API → Infrastructure → Application → Domain

- [ ] `docker-compose.yml` with SQL Server 2022

- [ ] `Program.cs` — Swagger, health check endpoint `/health`

- [ ] Serilog basic console logging

- [ ] Global exception middleware skeleton (catches and returns Problem Details)

- [ ] `.gitignore`, `README.md`

- [ ] Commit: `"Phase 0: Project skeleton"`



**Done when:** `docker-compose up` starts SQL Server, `dotnet run` starts API, `/health` returns 200.



--- 

### Phase 1: Core Domain + Incident Reporting (No Auth) ✅ COMPLETE

**Goal:** Create and retrieve incidents. No auth. No multi-tenancy. Just the core loop.



New concepts: Clean Architecture structure, EF Core code-first, first migration, MediatR, FluentValidation, CQRS in practice.



- [ ] BaseEntity.cs (Id, CreatedAt, UpdatedAt)

- [ ] Organization, Site, SiteArea entities

- [ ] CompanyType enum

- [ ] Incident entity + IncidentType, Severity, IncidentStatus enums

- [ ] ApplicationDbContext with DbSets

- [ ] IEntityTypeConfiguration per entity

- [ ] First EF migration

- [ ] CreateIncidentCommand + Handler + Validator

- [ ] GetIncidentsQuery + Handler (simple list)

- [ ] GetIncidentByIdQuery + Handler

- [ ] IncidentsController (POST, GET)

- [ ] ValidationBehaviour in MediatR pipeline

- [ ] Test: create incident via Swagger, retrieve it



**Done when:** POST an incident, GET it back. Stored in SQL Server.



---



### Phase 2: Incident Lifecycle — Status Machine ✅ COMPLETE

**Goal:** Incidents have an enforced lifecycle.



New concepts: State machine in domain, domain exceptions, PUT/PATCH endpoints, pagination, filtering.



- [ ] `Incident.TransitionTo(newStatus)` — domain method, enforces rules

- [ ] `InvalidStatusTransitionException`

- [ ] UpdateIncidentCommand + Handler

- [ ] UpdateIncidentStatusCommand + Handler

- [ ] AssignIncidentCommand + Handler

- [ ] PaginatedList<T> + update GetIncidents (page, size, filters)

- [ ] Filter by status, severity, type, site

- [ ] Commit: `"Phase 2: Incident status machine"`



**Done when:** Can't skip from Reported to Closed. Status transitions enforced.



---



### Phase 3: Corrective Actions ✅ COMPLETE

**Goal:** Incidents have corrective actions. CAs block incident resolution.



New concepts: One-to-many in EF, nested resources, cross-entity business rules.



- [x] CorrectiveAction entity + enums (Priority, CorrectiveActionStatus)
- [x] EF relationship + migration (nullable IncidentId FK, filtered index — ADR-011)
- [x] CRUD commands/queries for CAs (Create, Update, UpdateStatus, SoftDelete, GetList, GetById)
- [x] CorrectiveAction.TransitionTo() domain method with state machine + guard clauses
- [x] CorrectiveActionsController (POST, GET list, GET by id, PUT, PATCH /status, DELETE)
- [x] Unit tests — 10 for Incident.TransitionTo(), 12 for CorrectiveAction.TransitionTo() + new CA guard
- [x] Business rule: Incident → Resolved only if all CAs are Completed/Verified
- [x] GetIncidentByIdQuery includes CAs in response



**Done when:** CA blocks incident resolution. Verified in Swagger.



---



### Phase 4: Users & Authentication ✅ COMPLETE

**Goal:** Real users with JWT. Roles enforced on endpoints.



New concepts: JWT in .NET 8, role-based authorization, password hashing, ICurrentUserService.



- [x] User entity + UserRole enum (all 8 roles)

- [x] AuthController: Register (creates Organization + first Admin user), Login

- [x] JWT generation with claims: userId, tenantId, companyType, email, role

- [x] ICurrentUserService + implementation via IHttpContextAccessor

- [x] [Authorize] on all incident/CA endpoints. [AllowAnonymous] on AuthController.

- [x] ForbiddenAccessException → 403 in ExceptionHandlingMiddleware

- [x] Role restrictions: only SafetyOfficer/OrganizationAdmin can close incidents (handler-level guard, Phase 8 replaces with permissions layer)

- [x] ReportedById on Incident wired from ICurrentUserService.UserId (removed from request body)

- [x] AssignIncidentHandler validates assignee exists and is active in DB

- [x] 15 Application tests + 22 Domain tests all passing

- [x] EHS-45 logged: Architecture Spike — Concurrency, Consistency, Auditability & Traceability (to be resolved before Phase 5 code is written)



**Done when:** Must login to use API. Role restrictions enforced. ✅



---



### Phase 5: Multi-tenancy + Infrastructure Plumbing ✅ COMPLETE

**Goal:** Client orgs and contractor orgs are completely isolated. Redis wired with clean abstraction for future Dapr swap.

New concepts: EF Core Global Query Filters, TenantResolutionMiddleware, IDistributedCache, ICacheService abstraction.

- [x] RowVersion on BaseEntity + DbUpdateConcurrencyException → 409 (EHS-46)
- [x] CorrelationId middleware + Serilog enricher + IRequestContext (EHS-47)
- [x] TenantId on all entities + EF configs + migration (EHS-48)
- [x] ClientContractor entity + EF config + migration (EHS-49)
- [x] EF Core Global Query Filters (TenantId + IsDeleted) + TenantResolutionMiddleware (EHS-50)
- [x] Redis docker-compose + `IDistributedCache` (StackExchange.Redis) + `ICacheService` interface in Application + `DistributedCacheService : ICacheService` in Infrastructure + cache key conventions (EHS-51)
- [x] Elasticsearch + Kibana local Docker spike + SQL Full-Text comparison (EHS-52)
- [x] Phase 5 docs update (EHS-53)

**ICacheService note:** Handlers inject `ICacheService`, never `IDistributedCache` directly. At Phase 8, swapping `DistributedCacheService` → `DaprCacheService` is a single DI line change. See `architectural-gaps.md` Gap 17.

**Done when:** Two orgs' data completely isolated. Redis resolves from DI. `ICacheService` registered.



---



### Phase 6: Audit Logging

**Goal:** Every change recorded with who, what, when, before/after.



New concepts: EF Core SaveChanges interceptors, immutable audit tables, change tracking.



- [ ] AuditLog entity + migration

- [ ] AuditInterceptor: SaveChangesInterceptor captures entity changes as JSON

- [ ] GET /api/incidents/{id}/audit-log endpoint

- [ ] Commit: `"Phase 6: Audit logging"`



**Done when:** Every incident update creates an immutable audit record.



---



### Phase 7: Semantic Form Engine

**Goal:** AI-readable supplementary form engine. Tenant admins build custom forms.
Core typed entities (Incident, CorrectiveAction, etc.) stay typed — forms are supplementary only.

Full design: `semantic-form-engine-design.md`. Architecture decision: ADR-013.

- [ ] `FormTemplate` entity + EF config + JSON column for `List<FormFieldDescriptor>`
- [ ] `FormFieldDescriptor` with: `AiDescription`, `SemanticConcept`, `RegulatoryReference`,
      `IsComplianceCritical`, `EntityPropertyPath`, `MaxScore`, `ScoreMap`, `AllowedValues`
- [ ] `FormSubmission` entity + `ParentEntityId` + `ParentEntityType` + `Values` JSON +
      `TotalScore`, `MaxPossibleScore`, `ScorePercentage`, `ScoreRating`
- [ ] `IFormSemanticContextBuilder` + `SemanticFormContext` + `ToAiContextString()`
- [ ] `EntityPropertyRegistry` — developer-registered mappable paths (~10-20 total)
- [ ] Entity mapper — writes form field values to typed entity properties on submission
- [ ] `IsSystemTemplate` flag + `IndustryTag` + `StandardReference` + `ClonedFromTemplateId`
- [ ] Pre-built industry templates (8 templates with full `AiDescription` pre-populated):
      ISO 45001 Hazard ID, OSHA 1904 Incident Supplement, CDM 2015 Contractor Induction,
      NFPA 51B Hot Work Permit, OSHA 1910.146 Confined Space, OSHA 1910.147 LOTO,
      Pre-Task Safety Briefing (JSA/JHA), Environmental Spill Response
- [ ] CRUD API for FormTemplate (create, get, clone system template)
- [ ] Submit form endpoint (FormSubmission + score computation)
- [ ] Structured admin builder UI (React — add/configure/reorder fields + RJSF live preview)
- [ ] RJSF renderer (`@rjsf/core` + `@rjsf/shadcn-ui`) with custom EHS widgets
- [ ] Commit: `"Phase 7: Semantic form engine"`

**Done when:** Tenant admin can create a template with AI metadata, worker submits form,
AI can read semantic context via `IFormSemanticContextBuilder`, score computed on save.



---



### Phase 8: Caching + Infrastructure Resilience

**Goal:** Activate `ICacheService` in query handlers. Evaluate Dapr adoption. Wire resilience, health checks, and idempotency.

New concepts: Cache population/invalidation, Dapr sidecar evaluation, Polly resilience pipelines, structured health checks.

- [ ] Activate caching in query handlers via `ICacheService` (already wired in Phase 5):
      site/area lookups, user permission maps, org settings, aggregate counts
- [ ] Explicit cache invalidation on write (clear relevant keys in write handlers)
- [ ] Output caching on reference data GET endpoints (`[OutputCache]` attribute — Gap 18)
- [ ] **Dapr evaluation (anchor: Phase 13 Work Permit Workflow):** Prototype Dapr Workflow for
      the work permit multi-step approval chain (Gas Tester → Safety Officer → Supervisor, waits
      up to 2hrs, escalates on no-response). This is the primary justification for running the sidecar.
      If adopted: `DaprCacheService : ICacheService` (State) + Dapr pub/sub (Phase 9 foundation) come
      free from the same sidecar — wire both at this point.
      If rejected: `DistributedCacheService` stays, OutboxProcessorJob built in Phase 9.
      See ADR-015 for full building block decision map.
- [ ] **Evaluate .NET Aspire** as local dev orchestrator — starts API + Dapr sidecar + Redis + SQL Server + OTEL dashboard with one command (Gap 24). Solves Gaps 8, 16, 20 for free if adopted.
- [ ] Idempotency keys for create commands — `IIdempotentCommand` marker + `IdempotencyBehaviour` pipeline behaviour (Gap 21)
- [ ] Structured health checks: `/health/live` + `/health/ready` probes for DB and Redis (Gap 20)
- [ ] Polly resilience on all external `HttpClient` registrations (Gap 16)
- [ ] Commit: `"Phase 8: Caching, Dapr evaluation, and infrastructure resilience"`

**Done when:** Reference data reads hit `ICacheService` (not DB). Dapr adoption decision locked. Health probes return 200.



---



### Phase 9: Background Workers + Outbox Pattern + Expiry Notifications

**Goal:** Reliable notifications. Overdue CA detection. Expiry notifications for documents and certs.

New concepts: BackgroundService, PeriodicTimer, Outbox pattern, Polly on email HttpClient.

- [ ] OutboxMessage entity + migration
- [ ] OutboxProcessorJob: reads OutboxMessage table, sends emails (with Polly retry on SendGrid HttpClient), marks processed
- [ ] OverdueCaJob: finds CAs past DueDate, creates OutboxMessages
- [ ] Basic email delivery (SendGrid or SMTP)
- [ ] Expiry notification triggers (EHS-55): domain events for document/cert expiry within 30/60/90 days — fires pub/sub event → OutboxMessage → email
- [ ] Polly resilience on email + any external HttpClient (Gap 16 — if not done in Phase 8)
- [ ] Hangfire scheduler + dashboard at `/jobs` (Gap 19 — do before job count exceeds 2-3)
- [ ] If Dapr adopted at Phase 8: replace OutboxProcessorJob with Dapr pub/sub `[Topic]` subscribers
- [ ] Commit: `"Phase 9: Background workers, outbox, and expiry notifications"`

**Done when:** Overdue CAs trigger emails. Documents/certs expiring within 30 days trigger warnings. Jobs visible in Hangfire dashboard.



---



### Phase 10: Contractor Compliance + File Attachments

**Goal:** Expand contractor management with insurance, training, document compliance, and induction tracking.
       Incidents can have photo/document attachments.

New concepts: Azure Blob Storage, composite compliance scoring, document review workflows, site check-in model.

**File attachments:**
- [ ] Azure Blob Storage (local: Azurite emulator) + `IBlobStorageService` interface + implementation
- [ ] `IncidentAttachment` entity + `POST /api/incidents/{id}/attachments` + `GET /api/incidents/{id}/attachments`

**Contractor compliance (from competitive analysis — EHASoft SHEQ gap):**
- [ ] Insurance tracking on ClientContractor: provider, type, coverage amount, currency, start/expiry date (EHS-58)
- [ ] Contractor vendor rating: Bronze/Silver/Gold + Risk Level (Low/Medium/High) + Valid Until date (EHS-59)
- [ ] "Ok to Work" composite score — computed per contractor/employee from document + training + questionnaire status (EHS-56)
- [ ] RAMS document type + multi-stage document review workflow: Pending → Pending EHS Review → Reviewed/Rejected (EHS-57)
- [ ] Induction scheduling — site/day/time slots, contractor type → induction group mapping, booking calendar (EHS-60)
- [ ] Site check-in data model — who is on site, when (foundation for emergency muster and Security Desk) (EHS-61)
- [ ] Expiry tracking foundation (EHS-55): ExpiryDate field on Insurance, Training, Document entities. Feeds Phase 9 expiry notifications.

**Design pre-work (before coding):**
- [ ] Design tenant-configurable rules engine (Gap 15) — `WorkflowRule` table per tenant, `WorkflowRulesBehaviour` in MediatR pipeline. Required for "Ok to Work" gates on Work Permit.
- [ ] Design external integration resilience pattern (Gap 14) — if security provider badge check is part of any rule.

- [ ] Commit: `"Phase 10: Contractor compliance and file attachments"`

**Done when:** Contractor has insurance tracked, a composite "Ok to Work" score, and documents go through review workflow. Incident attachments upload to blob storage.



---



### Phase 11: Dashboard, Reporting + Scheduled Reports

**Goal:** Summary stats a safety manager would actually use. Automated scheduled reports. Job observability.

New concepts: Aggregate queries, Hangfire recurring jobs, scheduled email delivery.

- [ ] `GET /api/dashboard/summary` — total incidents by status/severity, overdue CAs, expiring documents
- [ ] `GET /api/reports/incidents` — filtered export
- [ ] Optimized queries (no N+1, use projections)
- [ ] **Scheduled automated reports** (EHS-62): configurable frequency (Daily/Weekly/Monthly/Yearly) to named recipients. Report types: Expiry Report, Contractor Summary, Induction Report, Overdue CA Summary. Hangfire cron + email template per report type.
- [ ] Hangfire dashboard locked behind admin auth at `/jobs` (Gap 19 — must be in before this phase)
- [ ] Elasticsearch evaluation vs SQL Full-Text at real data volumes (as per ADR-012 Elasticsearch decision)
- [ ] Commit: `"Phase 11: Dashboard, reporting, and scheduled reports"`

**Done when:** Safety manager can see live summary dashboard, export filtered reports, and configure automated weekly reports to their inbox.



---



### Phase 12: React Frontend

**Goal:** Working UI for core flows. Role-based views.



Screens:

1. Login / Register

2. Dashboard (role-aware — Safety Officer sees different data than Contractor Worker)

3. Incident list (table with filters)

4. Create incident form

5. Incident detail (with CAs, audit log)

6. Create corrective action

7. Form builder UI (Phase 7 engine) — basic

8. Site/Area management



**Done when:** Can login, see role-specific dashboard, create incident, add CA — all via UI.



---



### Phase 13: Work Permit Management

**Goal:** Full work permit lifecycle built on top of existing platform.

(Work permits are the most complex module in EHS — gas testing, isolations, PPE, multi-step approvals)

**Approval chain:** This is the anchor use case that drives Dapr adoption at Phase 8.
If Dapr adopted: use Dapr Workflow (durable, survives pod restarts, waits up to 2hrs for approver, escalates on no-response).
If Dapr rejected: custom state machine in SQL with PeriodicTimer polling — significantly more code.
See ADR-015 for full Dapr building block decisions and decision sequence.



---



### Phase 14: Document Management

**Goal:** Compliance document repository with expiry tracking.



---



### Phase 15: Compliance Audit Management

**Goal:** Conduct formal compliance audits, findings, corrective actions.



---



### Phase 16: Azure Deployment + CI/CD

**Goal:** Production-quality deployment. Live on Azure.



- [ ] Dockerfile for API

- [ ] Dockerfile for frontend

- [ ] Azure Container Registry

- [ ] Azure Container Apps

- [ ] Azure SQL Database

- [ ] Azure Cache for Redis

- [ ] Azure Key Vault for secrets

- [ ] GitHub Actions: build → test → push → deploy on merge to main

- [ ] Azure Application Insights



**Done when:** Push to main automatically deploys to Azure. App is live.



---



### Phase 17: AI-First Integration

**Goal:** Not "AI as a feature" — the platform built so AI agents can operate it natively.

Full strategic rationale: see [ai-first-strategy.md](ai-first-strategy.md)

**Domain AI features (original scope — kept):**
- **Incident Classification:** AI suggests Type and Severity from description text
- **Pattern Detection:** "You've had 5 similar incidents in 30 days — review X procedure"
- **Risk Scoring:** AI scores new incidents for likely severity escalation
- **Compliance Report Generation:** AI narrative summary for a date range (streaming)
- **Form Template Generator:** Describe what you need, AI generates a form template

**AI-Native Infrastructure (added scope):**
- **EHS MCP Server** — production MCP server exposing the full API surface as typed AI tools.
  Any AI assistant (Claude, Copilot, etc.) can call our platform directly.
  MCP tools map 1:1 to existing MediatR Commands/Queries — no new business logic.
- **AI Service Principal** — a user type representing an AI agent. All agent actions are
  fully attributed and audited like human actions.
- **Domain Event Subscriptions** — AI agents subscribe to Outbox events (incident created,
  CA overdue, permit expiring) and react without polling.
- **ERP/CRM Integration via AI** — AI orchestrator mediates between our MCP server and
  ERP MCP servers (SAP, Oracle EAM, Dynamics 365). Intent-based, not field-mapping-based.
  Work Permit (Phase 13) is the primary ERP integration bridge.

**Tech:** Claude API (Anthropic .NET SDK), `ModelContextProtocol` NuGet for MCP server,
streaming responses for reports. Runs in the same API process — no new infrastructure.

**Architecture decisions (locked — ADR-014):**
- `IAiSuggestionService` abstraction — provider-agnostic. Swap Anthropic/Groq/DeepSeek/Gemini/Ollama
  via `appsettings.json`. Application layer never knows which provider is active.
- AI suggestions constrained to typed enum values — hallucination structurally limited.
- AI Service Principal — all AI actions attributed, timestamped, paired with human decision in audit trail.
- Subscription tier gating — Free: no AI. Standard: classification only. Professional: full AI.
  Enterprise: unlimited + configurable provider.
- Just Culture language enforcement — all AI-generated text passes through a system prompt
  component enforcing blame-free, system-focused safety language.
- `RegulatoryContent` extensibility table — platform builds the shelf; customers/partners bring content.
  Designed to plug in regulatory MCP servers (OSHA eCFR, Wolters Kluwer) when they emerge.

**What NOT to build (locked — see ai-capabilities-research.md Section 7):**
- No autonomous permit approval, no autonomous severity changes
- No general-purpose regulatory Q&A chatbot
- No fully autonomous CA creation
- No computer vision monitoring
- No AI-generated ESG reports for public disclosure

Full rationale: `ai-first-strategy.md`, `ai-capabilities-research.md`, `ai-strategy-session-handoff.md`



---



## Development Environment Setup (To Do in Phase 0 Session)



### Prerequisites (Personal Machine)

- .NET 8 SDK: https://dotnet.microsoft.com/download/dotnet/8.0

- Docker Desktop: https://www.docker.com/products/docker-desktop

- VS Code or Visual Studio 2022 Community

- Git

- Node.js 20+ (for frontend phases)



### Local Run (after Phase 0)

```bash

docker-compose up -d # starts SQL Server (+ Redis from Phase 8)

dotnet ef database update # applies migrations

dotnet run --project src/EHSPlatform.API

# open https://localhost:{port}/swagger

```



---



## What MAISAAS Had That We Will Add in Later Phases



| MAISAAS Module | Our Equivalent | Phase |

|---|---|---|

| Work Permit Management | Phase 13 | Later |

| Document Management (with expiry) | Phase 14 | Later |

| Audit Management (compliance audits) | Phase 15 | Later |

| Training & Skills | Future | Much later |

| Check-in / Check-out (QR, badges) | Phase 10 (EHS-61) + Phase 13 QR | Added from competitive analysis |

| Vendor/Contractor Rating | Phase 10 (EHS-59) | Added from competitive analysis |

| Insurance Tracking | Phase 10 (EHS-58) | Added from competitive analysis |

| Induction Management (scheduling, groups) | Phase 10 (EHS-60) | Added from competitive analysis |

| "Ok to Work" Composite Score | Phase 10 (EHS-56) | Added from competitive analysis |

| Expiry Notification Engine | Phase 9 (EHS-55) | Added from competitive analysis |

| Scheduled Automated Reports | Phase 11 (EHS-62) | Added from competitive analysis |

| Minutes of Meeting | Probably skip | Skip |

| SCIM 2.0 / SSO | Future | Enterprise phase |

| Power BI integration | Future | Enterprise phase |

| RabbitMQ | Replaced by Outbox Pattern | Already decided |

| Quartz.NET | Replaced by .NET BackgroundService | Already decided |



## What We Have That MAISAAS Didn't



| Our Addition | Why |

|---|---|

| Clean Architecture + CQRS | Testable, maintainable, scalable |

| MediatR pipeline behaviors | Automatic validation/logging/audit on every request |

| Dynamic Form Engine | Their questionnaire, plus future integration with modules |

| ISO template library | Commercial differentiator (future) |

| AI integration | Pattern detection, classification, report generation |

| React SPA | Modern frontend vs Razor MVC |

| Outbox Pattern | Reliable notifications without RabbitMQ complexity |

| Domain logic in entities | Not anemic domain model like MAISAAS |

| Full test suite | MAISAAS had no tests |



---



## Session Resume Instructions (For Any AI Assistant)



1. Read this file completely before doing anything else

2. Read career-profile.md for developer context

3. Check which phases are marked complete (checkboxes above)

4. Ask the developer which phase they want to work on

5. **Never re-debate decided things** — all decisions in this document are locked

6. Remind the developer of anti-procrastination rules if they start scope-creeping

7. The developer practices on a **personal machine only** — not the company laptop

8. Goal for each session: end with committed, runnable code



**Current status: Phase 0 ✅ Phase 1 ✅ Phase 2 ✅ Phase 3 ✅ Phase 4 ✅ Phase 5 ✅ Phase 6 next.**
