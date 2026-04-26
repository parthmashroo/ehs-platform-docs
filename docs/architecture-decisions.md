# Architecture Decisions

This document records every significant architecture decision made for the EHS Platform.
Each decision includes what was decided, why, and what alternatives were considered.
These decisions are locked — do not revisit without a strong reason.

---

## ADR-001: Clean Architecture with 4 Layers

**Decision:** Structure the solution into 4 projects:
Domain, Application, Infrastructure, API.

**Why:**
- Each layer has one responsibility and one only
- Dependencies flow inward only — API → Infrastructure → Application → Domain
- Domain and Application have zero knowledge of databases, HTTP, or external services
- Switching database (SQL Server → PostgreSQL) only affects Infrastructure
- Switching API framework only affects API layer
- Forces testability — Application layer can be tested without a real database

**Alternatives considered:**
- 3-layer (Presentation/Business/Data) — rejected because business logic leaks into
  presentation and data layers over time
- MAISAAS-style flat layering — rejected because it caused tight coupling and made
  the system hard to change

**Layer rules (enforced by project references):**
- Domain → no dependencies on any other layer
- Application → depends on Domain only
- Infrastructure → depends on Application (and Domain through it)
- API → depends on Infrastructure only (not Application directly)

---

## ADR-002: CQRS with MediatR

**Decision:** All business operations are either Commands (writes) or Queries (reads),
handled by separate MediatR handlers. No shared handlers between reads and writes.

**Why:**
- Read models can be optimized independently from write models
- Every request goes through the MediatR pipeline — validation, logging, audit happen
  automatically without touching handler code
- Handlers are small, focused, and independently testable
- Adding a new feature = adding a new Command or Query. Nothing else changes.

**MediatR Pipeline (every request passes through this in order):**
1. ValidationBehaviour — runs FluentValidation on every command
2. LoggingBehaviour — logs command name and execution time
3. AuditBehaviour — records writes to AuditLog table
4. Handler — actual business logic

**Alternatives considered:**
- Traditional service layer (IIncidentService) — rejected because services grow large
  over time and become hard to maintain (MAISAAS WorkPermitService had 14 injected
  dependencies — a clear warning sign)

---

## ADR-003: Multi-tenancy via EF Core Global Query Filters

**Decision:** Single database, all tenants share tables. Every tenant-scoped entity has
a TenantId column. EF Core Global Query Filters automatically append
WHERE TenantId = @currentTenantId to every query.

**Why:**
- Simpler than separate databases per tenant — one migration, one connection string
- Automatic — developers cannot forget to filter by tenant (the filter is always on)
- Cost effective — no per-tenant database provisioning needed
- Scales well for the number of tenants we expect

**MAISAAS comparison:**
- MAISAAS filtered by CompanyId manually in every service method
- One missed WHERE clause = data leak across tenants
- Our approach makes that class of bug impossible

**Alternatives considered:**
- Separate database per tenant — rejected, too complex and expensive at this stage
- Separate schema per tenant — rejected, migration complexity not worth it

---

## ADR-004: CompanyType — Client vs Contractor

**Decision:** Every Organization has a CompanyType enum: Client (1) or Contractor (2).
This distinction is fundamental and drives the entire authorization model.

**Why:**
- Client = site owner. Has authority, oversight, approval power.
- Contractor = sends workers to client sites. Executes work, submits documents.
- These two types see different data, have different roles, different workflows
- Baking this into the data model from day one prevents retrofitting later

**Impact:**
- JWT token includes CompanyType claim
- TenantResolutionMiddleware extracts it on every request
- Role enum covers both sides (OrganizationAdmin, SafetyOfficer for Client side;
  ContractorAdmin, ContractorWorker for Contractor side)

---

## ADR-005: JWT Authentication Inside the API

**Decision:** JWT tokens are generated and validated inside the API project itself.
No separate authentication server (no IdentityServer, no Azure AD B2C in early phases).

**Why:**
- Simpler — one less service to run, deploy, and maintain
- Sufficient for early phases — we control the token contents completely
- MAISAAS used IdentityServer4 (now EOL) which added significant complexity

**JWT Claims included:**
- userId, tenantId, companyType, email, role

**Future:** SCIM 2.0 / Azure AD / Okta SSO added in a later phase for enterprise customers.

---

## ADR-006: Outbox Pattern for Notifications

**Decision:** When a command executes, domain events are saved to an OutboxMessage table
in the same database transaction as the business data. A background worker reads and
processes them (sends emails).

**Why:**
- Guaranteed delivery — if the email server is down, the message stays in OutboxMessage
  and retries later. No events lost.
- No distributed transaction needed — business data and event are saved atomically
- Simple — no message broker infrastructure required in early phases

**MAISAAS comparison:**
- MAISAAS used RabbitMQ with durable: false and autoAck: true
- Messages were lost on RabbitMQ restart and on handler failure
- Our approach is more reliable with less infrastructure

**Future:** Azure Service Bus replaces the outbox worker when scale requires it (Phase 15+).

---

## ADR-007: Soft Deletes Everywhere

**Decision:** No hard deletes. Every entity has IsDeleted (bool) flag.
Deleted records stay in the database but are filtered out of all queries.

**Why:**
- EHS is a compliance domain — data must never be permanently destroyed
- Audit trail requires the full history including "deleted" records
- Regulatory requirements in safety-critical industries demand full traceability

---

## ADR-008: Single React App with Role-Based Views

**Decision:** One React frontend application. Not separate apps for Client and Contractor.
The app reads the user's role and CompanyType from the JWT on login and renders
the appropriate dashboard, navigation, and screens.

**Why:**
- One codebase to maintain, one deployment
- Role-based rendering is straightforward in React
- If needed later: client.app.com and contractor.app.com subdomains via DNS pointing
  at the same deployed app — zero extra code

---

## ADR-009: Site Hierarchy — Organization → Site → SiteArea

**Decision:** Incidents and work permits link to a Site (and optionally a SiteArea),
not free text location fields.

**Why:**
- Free text locations are unsearchable and inconsistent
  ("Zone A", "zone a", "Zone-A" are treated as different locations)
- Structured hierarchy enables location-based filtering and reporting
- Matches how real EHS systems work — incidents belong to a physical location

---

## ADR-010: Technology Stack (Locked)

| Concern | Decision | Reason |
|---|---|---|
| Framework | .NET 8 Web API | LTS, modern, production ready |
| Architecture | Clean Architecture | Testable, maintainable, scalable |
| Pattern | CQRS + MediatR | Pipeline behaviors, separation of concerns |
| ORM | EF Core 8 | Code-first, migrations, interceptors for audit |
| Validation | FluentValidation | Declarative, pipeline-integrated |
| Logging | Serilog | Structured logging, multiple sinks |
| API Docs | Swashbuckle (Swagger) | .NET 8 compatible, industry standard |
| Frontend | React 18 + TypeScript | Modern, component-based, type-safe |
| Database (local) | SQL Server 2022 | Already installed, matches Azure SQL |
| Database (cloud) | Azure SQL | Managed, scalable |
| CI/CD | GitHub Actions | Free, integrated with repo |
| Hosting | Azure Container Apps | Serverless containers, cost effective |

---

## ADR-011: CorrectiveAction Source Linking — Separate Nullable FKs + ICorrectiveActionSource

**Decision:** A CorrectiveAction links to its source entity (Incident, AuditFinding, etc.)
via separate nullable FK columns — one per source type. A shared `ICorrectiveActionSource`
interface enforces the "all CAs must be closed before source can resolve" rule uniformly
across all source types. Filtered indexes (WHERE SourceId IS NOT NULL) are used per FK column.

```csharp
// Entity
public class CorrectiveAction : BaseEntity
{
    public Guid? IncidentId { get; set; }       // Phase 3
    public Guid? AuditFindingId { get; set; }   // Phase 15 — add when needed
    // New source types added as new nullable columns per phase
}

// Domain contract — every source entity implements this
public interface ICorrectiveActionSource
{
    Guid Id { get; }
    bool CanResolve(IEnumerable<CorrectiveActionStatus> caStatuses);
}

// Filtered index per source (added in EF config)
// CREATE INDEX IX_CA_Incident ON CorrectiveActions(TenantId, IncidentId)
// WHERE IncidentId IS NOT NULL
```

---

**The questions that drove this decision (full reasoning trail):**

*Q: Can a CA exist without any source — standalone?*
Yes. Management-driven CAs, proactive risk mitigations, and policy-driven actions have
no parent entity. IncidentId must be nullable.

*Q: Can one source have many CAs?*
Yes — one-to-many. A single incident can have 5, 10, or 20 CAs. A compliance audit
finding commonly spawns multiple CAs. This is the expected primary access pattern.

*Q: Can one CA link to multiple sources simultaneously?*
No. A CA has exactly one origin — or none (standalone). Many-to-many is not the domain.

*Q: Which modules will realistically generate CAs across all 17 phases?*
Incident (Phase 3), Work Permit violations (Phase 13), Compliance Audit findings (Phase 15),
Risk Assessments (future), Non-Conformances/ISO (future), Site Inspections (future).
Bounded set of 5–7 known source types. Not open-ended. Not user-defined.

*Q: What does the combined CA status rule look like?*
A source entity can only transition to Resolved/Closed when ALL its linked CAs are
Completed or Verified. This rule is identical across all source types — hence the
ICorrectiveActionSource interface captures it once and all source entities implement it.

---

**Scale analysis (informed this decision):**
- Realistic at full SaaS scale: 7 modules × 100k records × 5 CAs = 3.5M rows per tenant
- At 100 tenants: ~350M rows in CorrectiveActions table
- With TenantId as leading column + filtered index per nullable FK:
  "Get all CAs for incident X" hits a filtered index covering only non-null IncidentId rows
  — dramatically smaller index, O(log n) lookup regardless of total table size
- "Can this incident resolve?" query uses EXISTS (SELECT TOP 1) — exits on first open CA,
  sub-millisecond even at 350M rows
- Adding a new nullable FK column at 350M rows: instant DDL, SQL Server does not rewrite data
  for nullable columns with no default

---

**All four options considered:**

| Option | Description | Verdict |
|---|---|---|
| A (chosen) | Separate nullable FKs per source type | Integrity + performance + bounded growth |
| B | Polymorphic: SourceEntityType (string) + SourceEntityId (Guid) | No FK constraint — unacceptable in compliance domain. String index covers all rows, no filtered optimization. Cross-module reporting needs string GROUP BY. |
| C | TPH inheritance (IncidentCA, AuditCA subclasses) | Same DB structure as A but with EF inheritance ceremony. Benefit marginal for 5-7 known types. Rejected — complexity not justified. |
| D | Junction/link table (many-to-many) | Solves a problem we don't have (one CA, many sources). Extra JOIN on every query. Rejected. |

**Why Option B was rejected at scale specifically:**
Filtered indexes are not possible on polymorphic columns — the index must cover all 350M rows.
String comparison on SourceEntityType adds overhead on every query. Cross-module reporting
("CAs by source module") requires GROUP BY on a string column with no referential integrity.
In a compliance domain where audit trail correctness is legally significant, an unverifiable
Guid → string pair is unacceptable.

**Why Option A's extensibility score (6/10 raw) is acceptable:**
Adding a new source type costs: one nullable column, one filtered index, one migration
(instant DDL), ~30-50 lines of application code (command update, query filter, validator).
The ICorrectiveActionSource interface means the "all CAs done" guard is written once and
tested once — not reimplemented per source. Real extensibility cost is predictable and low.

---

**Future implications to remember:**
- Phase 5 (multi-tenancy): TenantId MUST be the leading column in all CA indexes
  or EF Global Query Filters will cause full table scans
- Phase 11 (reporting): Cross-module CA dashboard may need a nightly pre-aggregated
  summary table — evaluate at Phase 11, not before
- Phase 16 (Azure): Consider table partitioning by TenantId if tenant count exceeds 500
- Soft delete accumulation: Archive CAs older than 2 years to CorrectiveActions_Archive
  in Phase 16 to keep active table and indexes lean

---

## What We Deliberately Did NOT Do (and Why)

| Rejected Approach | Why Rejected |
|---|---|
| Separate auth server (IdentityServer) | Complexity not needed in early phases |
| RabbitMQ | Outbox pattern is simpler and more reliable for our scale |
| Separate database per tenant | Too expensive and complex |
| Two separate frontend apps | One app with role-based views is simpler |
| Hard deletes | Compliance domain requires full audit trail |
| Quartz.NET | .NET BackgroundService is sufficient, less infrastructure |
| BinaryFormatter (Redis) | Obsolete in .NET 5+, use System.Text.Json instead |

---

## MAISAAS Lessons Applied

The EHS Platform is inspired by MAISAAS (a real ERP system). These are the specific
problems in MAISAAS that influenced our architecture decisions:

| MAISAAS Problem | Our Solution |
|---|---|
| Manual tenant filtering — one missed WHERE = data leak | EF Core Global Query Filters |
| Fat service classes (14 injected dependencies) | CQRS handlers — one handler, one job |
| No global query filter | Global Query Filters on all entities |
| RabbitMQ with durable:false, autoAck:true | Outbox Pattern |
| No structured logging (flat text files) | Serilog with structured output |
| JWT ValidateLifetime = false | Proper JWT validation |
| Weak signing key ("secret") in config | Azure Key Vault (Phase 16) |
| BinaryFormatter for Redis | System.Text.Json |
| No tests | Full xUnit test suite planned |
| Files stored on local disk | Azure Blob Storage (Phase 10) |
