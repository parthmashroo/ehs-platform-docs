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

## ADR-012: Concurrency, Consistency, Auditability, and Traceability Strategy

**Context:** EHS-45 (Architecture Spike). Finalised after review against 4 independent AI
analyses and an LLM council synthesis. All decisions locked unless marked **OPEN**.

---

### Gap 1 — Write-Write Concurrency (Lost Updates)

**Decision:** EF Core optimistic concurrency via `RowVersion` on `BaseEntity`. Exposed via
**both** ETag headers (authoritative contract) and `rowVersion` field in response body (convenience).

**API contract (locked):**
- GET response: `ETag: W/"<base64-rowVersion>"` header AND `rowVersion` field in JSON body
- PUT/PATCH on state-transition endpoints (Close, Reopen, Approve): `If-Match` is **mandatory**
- PUT/PATCH on non-critical field edits (description, title): `If-Match` optional
- Missing `If-Match` on mandatory endpoints → `428 Precondition Required`
- Stale `If-Match` → `412 Precondition Failed`
- `DbUpdateConcurrencyException` (domain-level conflict) → `409 Conflict`

**Why ETags, not body-only:** HTTP caching infrastructure (CDN, proxies, conditional GETs)
understands ETags natively. The 412 vs 409 distinction is forensically significant in compliance
audits — "client sent stale token" is a different audit event from "domain state makes this impossible."

**Why not pessimistic locking:** Holds DB connections across HTTP request lifecycle, kills
throughput under load, deadlock risk rises non-linearly with concurrent users.

**Why not event sourcing:** AuditLog (Gap 3) satisfies all traceability needs. Deferred permanently.

**Implementation:** EHS-46, Phase 5 first step.

---

### Gap 2 — Read-Write Inconsistency (Stale Reads)

**Decision:** Direct DB reads until Phase 7. Redis for caching from Phase 7 onward.
`ICacheService` abstraction wired in Phase 5 via `DistributedCacheService` (backed by StackExchange.Redis) — no caching logic yet. Dapr State building block is the Phase 8 evaluation candidate to replace `DistributedCacheService`.

**Scale threshold (corrected after multi-AI review):** "Direct DB reads are sufficient"
expires at approximately 250 concurrent active tenants under the synchronised shift-change
burst pattern. Each worker check-in costs ~7 DB round-trips.

| Active tenants | Peak DB ops/sec at shift change | Verdict |
|---|---|---|
| Up to 50 | ~700 | Comfortable — direct DB fine |
| 100 | ~1,400 | Manageable |
| ~250 | ~3,500 | Direct reads becoming a liability |
| 500+ | ~7,000+ | Redis required |

**Phase 5 (EHS-51):** Wire Redis in `docker-compose.yml` + register `IDistributedCache` (StackExchange.Redis) in DI.
Add `ICacheService` interface in Application layer. Add `DistributedCacheService : ICacheService` in Infrastructure.
No caching logic yet. Establish cache key convention: `tenant:{tenantId}:{entity}:{id}`.
Handlers inject `ICacheService` — never `IDistributedCache` directly.

**Why ICacheService abstraction:** Dapr State building block is a Phase 8 candidate to replace `DistributedCacheService`.
With this abstraction, the swap is a single DI line change. Zero handler changes required.

```
Phase 5:              services.AddScoped<ICacheService, DistributedCacheService>();
Phase 8 (if Dapr):    services.AddScoped<ICacheService, DaprCacheService>();
                      ↑ One line change. Every handler is unaffected.
```

**FusionCache:** Rejected. Adds L1 in-process + stampede protection but introduces an abstraction layer
that obscures what Redis is doing — reducing learning value. `ICacheService` + Dapr State achieves
equivalent cross-pod benefits at Phase 8 without the intermediate dependency.

**Phase 7:** Activate caching in query handlers via `ICacheService`. Start with reference data and aggregate counts.
When horizontal scaling is needed (2+ API instances), the same Redis instance is already shared —
zero code changes, only a connection string change for production.

**Phase 8 (Dapr evaluation):** Implement `DaprCacheService : ICacheService` as the Dapr State building block.
Decision made at Phase 8 based on actual multi-pod evidence. Not locked now.
See `architectural-gaps.md` Gap 1 and Gap 17 for full reasoning.

**Caching scope — critical rule, never cache live operational state with a TTL:**

| Cache this ✅ | Do NOT cache this ❌ |
|---|---|
| Site / SiteArea lookup data | Live incident or permit status |
| Organisation settings | Corrective action state |
| User permission maps | Any record where stale = safety risk |
| Dashboard aggregate counts (explicit invalidation on write) | |

**Redis infrastructure path:**

| Stage | Setup | Cost |
|---|---|---|
| Dev / Learning | Local Docker or Podman | Free forever |
| Early production | Self-hosted Redis on a £5-10/mo VM alongside the app | £5-10/mo |
| Scale (250+ tenants) | Azure Cache for Redis Basic/Standard | £40-150/mo |
| If Dapr adopted | Dapr State component YAML — swap provider without code change | Same cost |

**Why not CQRS projections / separate read store:** Redis covers us to 1,000+ tenants.
Separate read infrastructure is not justified at our scale.

**Why not Azure SQL Read Replicas:** Requires Business Critical SQL tier — ~4× the cost.
Redis achieves the same outcome for the cost of a Docker container.

**Implementation:** EHS-51 (Phase 5 plumbing + ICacheService abstraction), Phase 7 (actual caching logic), Phase 8 (Dapr evaluation).

---

### Gap 3 — Audit Trail (Who Changed What, When)

**Decision:** Three separate, immutable audit tables. All marked as Azure SQL Ledger Tables.

| Table | Written by | Captures | Primary reader |
|---|---|---|---|
| `EntityAuditLog` | EF `SaveChangesInterceptor` | Every successful mutation: EntityName, EntityId, Action, OldValues (JSON), NewValues (JSON), ChangedById, ChangedAt, CorrelationId | Compliance auditor, support |
| `SecurityAuditLog` | MediatR `SecurityAuditBehavior` | 401s, 403s, cross-tenant access attempts | Security team, incident response |
| `BusinessRuleAuditLog` | MediatR `SecurityAuditBehavior` | Domain rule violations, invalid state transitions, concurrency conflicts (409s) | Compliance auditor, app support |

**What NOT to audit:** Validation failures (400s — missing fields, bad format). Noise, not compliance.

**Azure SQL Ledger Tables:** All three tables created with `LEDGER = ON` in the Phase 6 migration.
Creates cryptographic digests in Azure Storage — makes post-commit tampering provably detectable.
Half a day of additional work. Negligible cost.

**Separate DbContext for security/business-rule audit:** Written via a dedicated `IAuditDbContext`
with its own transaction, independent of the main application context. Ensures security audit records
persist even when the main transaction rolls back.

**Rate-limit 401 logging:** Max 1 row per IP per minute after 10 failed attempts. Prevents
log-flood DoS from credential-stuffing bots.

**OPEN — Audit table structure at scale:** EntityAuditLog at full scale (17 modules, 1,000 tenants,
5 years) could reach 500M+ rows. Two options under active discussion, not yet decided:

- **Option A (current plan):** Three shared tables for all 17 modules. Managed via indexes on
  `(TenantId, EntityName, EntityId, ChangedAt)` and date-based archiving.
- **Option B:** Three tables per master entity — `IncidentEntityAuditLog`, `CorrectiveActionEntityAuditLog`,
  etc. Smaller tables, cleaner per-module retention policies, ~51 audit tables at 17 modules.

Revisit at Phase 6 with the full module list known. Start with Option A. Split only if
measurements at that point justify it.

**Implementation:** Phase 6.

---

### Gap 4 — Issue Replay and Traceability

**Decision:** Serilog structured logs + three audit tables + CorrelationId as the shared join key.
No event sourcing. No distributed tracing infrastructure.

**CorrelationId wiring (EHS-47):**
- `CorrelationIdMiddleware` checks for an inbound `traceparent` header (W3C Trace Context standard).
  Uses it if present and valid. Generates a new GUID if absent.
- `LogContext.PushProperty("CorrelationId", id)` wraps the entire request — every Serilog entry
  for that request automatically carries it. No changes needed to individual log statements.
- `IRequestContext` interface defined in Application layer, implemented in Infrastructure via
  `IHttpContextAccessor`. Phase 9 background workers implement it separately — reading CorrelationId
  from the OutboxMessage row being processed. Do NOT inject `IHttpContextAccessor` directly into handlers.
- CorrelationId always emitted on the response header.

**How to reconstruct any incident:**
- By record: `SELECT * FROM EntityAuditLog WHERE EntityId = @id ORDER BY ChangedAt`
- By user: `SELECT * FROM EntityAuditLog WHERE ChangedById = @userId ORDER BY ChangedAt`
- By request: `SELECT * FROM EntityAuditLog WHERE CorrelationId = @id`
- Failed attempts: `SELECT * FROM SecurityAuditLog WHERE Resource = 'Incident:{id}'`
- Cross-reference all with Serilog logs via CorrelationId for full execution trace.

**CausationId (Phase 9 note):** When Outbox worker starts chaining operations (request → domain
event → downstream action), add `CausationId` alongside `CorrelationId`. CorrelationId = originating
request. CausationId = immediate parent operation. Design at Phase 9, not before.

**Why not event sourcing:** Entity row is always current state. AuditLog is a secondary forensic
record. These are different roles. AuditLog answers "what changed?" — not "what is the state?"

**Implementation:** EHS-47, Phase 5.

---

### Additional Decision — Elasticsearch

**Decision:** Elasticsearch adopted as the future search and reporting layer, introduced progressively.
SQL Server Full-Text Search used first in production. Both run locally for learning.

**Path:**

| Stage | What | Cost |
|---|---|---|
| Now (EHS-52 spike) | Local Docker/Podman — `elasticsearch:8.x` + `kibana:8.x`. Learning and comparison only. | Free |
| Phase 5–10 (production) | SQL Server Full-Text Search only. Already in SQL Server, zero new infrastructure. | Free |
| Phase 11+ (production) | Evaluate Elasticsearch when SQL Full-Text hits limits. Introduced via Phase 9 Outbox worker. | Free self-hosted → £10-20/mo VM → managed when revenue justifies |

**SQL Full-Text vs Elasticsearch:**

| | SQL Full-Text | Elasticsearch |
|---|---|---|
| Setup | Zero — already in SQL Server | Docker locally; service in production |
| Fuzzy matching | Weak | Excellent |
| Relevance ranking | Basic | Sophisticated |
| Faceted aggregations at scale | Slow | Fast |
| Dual-write required | No | Yes |
| Right phase | Phase 1–10 | Phase 11+ evaluation |

**Free alternatives if managed cost is not yet justified:** Meilisearch, Typesense, OpenSearch —
all self-hostable, open source. Elasticsearch concepts transfer directly.

---

### Additional Decision — Rate Limiting

**Decision:** Per-tenant rate limiting deferred as documented tech debt.

Built-in `.NET RateLimiting` middleware (~30-50 lines, free) handles this at code level.
Azure API Management handles it at infrastructure level (additional cost). Deferred until
a misbehaving tenant causes a production incident. Not a Phase 5 blocker.

---

### Additional Decision — Idempotency Keys

**Decision:** Idempotency keys for critical commands deferred to Phase 8.

Platform is a mobile-responsive React web app (Phase 12) — not a native iOS/Android app.
Browser HTTP clients have more predictable retry behaviour. Design at Phase 8 when the
full API surface is known.

---

### Implementation Tickets (Sequenced)

| Ticket | Description | Phase |
|---|---|---|
| EHS-46 | RowVersion on BaseEntity + ETag/If-Match contract + 412/428/409 handler | Phase 5, step 1 |
| EHS-47 | CorrelationId: Serilog enricher + IRequestContext + stamp on all audit rows | Phase 5 |
| EHS-48 | TenantId on all existing entities + combined migration (with EHS-46 RowVersion) | Phase 5 |
| EHS-49 | ClientContractor entity + EF config + migration | Phase 5 |
| EHS-50 | EF Core Global Query Filters + TenantResolutionMiddleware | Phase 5 |
| EHS-51 | Redis `IDistributedCache` + `ICacheService` interface (Application) + `DistributedCacheService` (Infrastructure) + docker-compose + cache key conventions | Phase 5 |
| EHS-52 | Elasticsearch + Kibana local Docker/Podman setup + SQL Full-Text comparison spike | Phase 5 (learning) |
| EHS-53 | Phase 5 docs update | Phase 5, last step |
| Phase 6 | EntityAuditLog + SecurityAuditLog + BusinessRuleAuditLog + Ledger Tables | Phase 6 |
| Phase 7 | Redis caching: reference data + aggregate counts, explicit invalidation in write handlers | Phase 7 |
| Phase 8 | Idempotency keys + rate limiting | Phase 8 |
| Phase 9 | Outbox + domain events + Elasticsearch indexing + CausationId | Phase 9 |
| Phase 11 | Elasticsearch production evaluation vs SQL Full-Text at real load | Phase 11 |

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
| FusionCache | Adds L1 in-memory + stampede protection but obscures Redis behaviour — reducing learning value and adding an intermediate abstraction. `ICacheService` + Dapr State (Phase 8) achieves cross-pod benefits at the right time without FusionCache. |
| Pessimistic DB locking | Holds connections across HTTP lifecycle, deadlocks under load |
| Broad TTL caching of live operational data | Stale incident/permit status is a safety risk in compliance domain |
| CQRS separate read store (current phases) | Redis covers us to 1,000+ tenants; separate read store is not justified yet |
| Event sourcing | AuditLog + RowVersion provides 95% of the benefit at 10% of the complexity |
| Native iOS/Android mobile app | React web app with mobile-responsive design covers Phase 12 needs |

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

---

## ADR-013: Semantic Form Engine Architecture (Phase 7)

**Context:** The original Phase 7 design used an EAV (Entity-Attribute-Value) pattern —
`FormTemplate → FormSection → FormField → FormSubmission → FormFieldAnswer`. This was
replaced in May 2026 after identifying that the EAV pattern has the same AI-readability
limitation as all form-engine-based EHS competitors. Full design: `semantic-form-engine-design.md`.

**Decision:** Three-layer semantic form engine replacing the EAV design.

**Layer 1 — FormTemplate with JSON column:**
`FormTemplate` stores `List<FormFieldDescriptor>` as a single EF Core 8 JSON column.
No normalised field-per-row table. Each `FormFieldDescriptor` carries:
- `AiDescription` — machine-oriented explanation for LLM consumption (not the display label)
- `SemanticConcept` — dot-notation namespace e.g. `"ContractorCompliance.InductionStatus"`
- `RegulatoryReference` — e.g. `"HSE CDM 2015 Regulation 12"`
- `IsComplianceCritical` — flags fields that determine compliance pass/fail
- `EntityPropertyPath` — optional path to write value back to a typed domain entity
- `MaxScore` / `ScoreMap` — scoring per field for audit/inspection scoring

**Layer 2 — FormSubmission envelope:**
Stores `Dictionary<string, object?> Values` as JSON. Always carries `FormTemplateId` +
`FormTemplateVersion` (pinned at submission time) + `ParentEntityId` + `ParentEntityType`
to anchor supplementary data to the typed domain. Computed score fields on save.

**Layer 3 — IFormSemanticContextBuilder:**
Application-layer service that joins submission values to field descriptors and produces
a structured semantic context string for LLM consumption. AI never sees `field_23: "yes"`.
It sees field name, AiDescription, allowed values, compliance flag, and the submitted value
with a compliance verdict. Fully unit testable — no HTTP, no LLM calls.

**EntityPropertyRegistry:**
Developer-maintained dictionary of ~10–20 paths where form field values should also be
written back to typed entity properties. Validated at template save time — invalid paths
fail immediately, not silently at submission. Tenant admins can only map to registered paths.

**Form Builder Library:**
`@rjsf/core` (React JSON Schema Form, Apache 2.0) as the renderer. Custom widgets replace
RJSF defaults for EHS-specific field types (toggle, camera capture, signature, severity picker).
Phase 7 builder: structured admin page (add/configure/reorder + live RJSF preview). Future
upgrade path to WYSIWYG canvas (Joyfill-quality) without any data model changes.
SurveyJS explicitly rejected: `survey-creator` (builder) is commercial ~$573/dev/year.
Form.io explicitly rejected: OSL-3.0 licence covers SaaS deployment.

**Pre-built industry templates:**
Platform ships a library of `IsSystemTemplate = true` templates with all `AiDescription`,
`SemanticConcept`, and `RegulatoryReference` fields pre-populated. Tenants use as-is or clone.
Starter pack: ISO 45001 Hazard ID, OSHA 1904 Incident Supplement, CDM 2015 Contractor Induction,
NFPA 51B Hot Work Permit Checklist, OSHA 1910.146 Confined Space, OSHA 1910.147 LOTO, and more.

**Why not EAV (original design):**
- EAV produces `field_23: "yes"` — opaque to LLMs without additional context
- Same AI-readability limitation that affects SafetyCulture, Intelex, Cority configurable modules
- JSON column with `AiDescription` per field gives LLMs semantic context at no extra query cost

**Why not OWL/RDF ontology:**
`SemanticConcept` dot-notation gives ~80% of the value of a full OWL URI at 5% of the complexity.
LLMs understand dot-namespaced concepts without dereferencing a URI. Full OWL is a future
optimisation if data volume justifies it. See `semantic-form-engine-design.md` for full reasoning.

---

## ADR-014: AI-First Architecture (Phase 17)

**Context:** Finalised during AI strategy review, May 2026. All 7 AI capability gaps in the
EHS market were reviewed and verdicts locked. Full research: `ai-capabilities-research.md`.
Full strategy: `ai-first-strategy.md`. All verdicts: `ai-strategy-session-handoff.md`.

**Core Posture — AI Advises, Human Decides. Always.**
Our AI never makes decisions, never approves documents, never changes severity autonomously,
never creates corrective actions without human confirmation. AI surfaces patterns from typed
historical data. All actions in the system are taken by named humans. This is the correct
posture for a safety-critical compliance domain and satisfies Morgan Lewis (Feb 2026):
"AI outputs are advisory rather than determinative."

**IAiSuggestionService — Provider Abstraction:**
Application layer defines the interface. Infrastructure implements per provider.
Providers: Anthropic Claude (default), Groq, DeepSeek, Gemini, Ollama (local dev).
Swap provider via `appsettings.json` — no application code changes.
Development: Ollama (free, no API calls). Production: Groq/Haiku for cheap tasks, Sonnet for complex.
AI suggestions are constrained to our typed enum values — hallucination is structurally limited.

**AI Service Principal — Full Audit Attribution:**
Every AI action is attributed to a named AI service principal (not "system"), recorded before
the human sees it, and paired with the human's accept/override decision in the same audit trail.
Regulators and legal proceedings get a complete record: "AI suggested X, named human overrode to Y."

**Subscription Tier Gating:**
AI features are gated by tenant subscription tier. Free tenants: zero AI calls.
Single check in each AI handler: `if (!features.IsAiEnabled) return NotAvailable()`.
Per-tenant monthly quota tracked. AI cost scales with revenue, not ahead of it.

**Just Culture Language Enforcement:**
All AI-generated text (CA descriptions, notifications, investigation templates, pattern reports)
passes through a Just Culture system prompt component. "PPE was not worn" not "worker forgot PPE."
System-focused language encourages reporting; blame language suppresses it. One system prompt
addition applied to every AI text generation call.

**What NOT to build (locked):**
- No autonomous permit approval (legal document — named human required)
- No autonomous severity changes (determines regulatory reporting threshold)
- No general-purpose regulatory Q&A chatbot (hallucination + legal liability)
- No fully autonomous CA creation (accountability implications)
- No computer vision monitoring (GDPR Article 9, workforce resistance, wrong domain)
- No AI-generated ESG reports for public disclosure (material disclosure, executive sign-off required)

**ERP Integration — Tiered Strategy:**
Tier 1 (Phase 13): Screenshot/PDF upload + AI extraction. Operator uploads ERP work order;
AI extracts work order number, asset, description, dates; pre-fills permit. Works with any ERP.
Tier 3 (Phase 17): Clean REST API + webhooks for iPaaS tools (MuleSoft, Azure Logic Apps, n8n).
Tier 5 (Phase 17): MCP orchestration when SAP/Dynamics MCP servers mature (12-18 months).
Tier 4: Direct ERP API adapters as professional services engagements, not product features.

**RegulatoryContent extensibility hook:**
`RegulatoryContent` table stores jurisdiction, standard, clause, official URL, and trigger condition.
Platform builds the shelf; enterprise customers populate their jurisdiction's requirements.
When regulatory content MCP servers emerge (Wolters Kluwer, OSHA eCFR API, EU EUR-Lex),
our architecture plugs them in with no structural changes. We build the shelf; partners bring content.

**Provider cost strategy:**
| Task | Provider | Approx cost |
|---|---|---|
| Incident classification | Groq Llama / Haiku | ~$0.001/call |
| CA suggestions | Groq / Haiku | ~$0.002/call |
| Pattern detection | Sonnet | ~$0.15/report |
| Report generation | Sonnet | ~$0.05/report |
| Per tenant per month | — | ~$3–8 |

Full detail: `ai-first-strategy.md` and `ai-strategy-session-handoff.md`.
