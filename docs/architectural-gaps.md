# EHS Platform — Architectural Gaps Register

> Identified: May 2026, pre-Phase-6 architectural challenge review.
> Purpose: Capture known gaps before the codebase grows too large to address them cheaply.
> Updated when a gap is resolved, deferred, or phased.

---

## How to Read This

| Status | Meaning |
|---|---|
| ✅ Resolved | Decision made, implementation path clear |
| 📅 Deferred | Will address at a specific phase — decision made |
| 🔄 Active | Needs action now or next phase |
| ⬜ Open | Identified, no decision yet |

---

## Quick Summary Table

| # | Gap | Status | When | Effort |
|---|---|---|---|---|
| 1 | Dapr (distributed building blocks) | ✅ Phase 8 candidate — not locked | Phase 8 evaluation | Medium |
| 2 | NetArchTest (architecture enforcement) | 🔄 Do now | Phase 5 or 6 | Half day |
| 3 | API Versioning | 📅 Deferred | Before Phase 12 | One day |
| 4 | DB Migration Strategy (production) | 📅 Deferred | Before Phase 16 | One day |
| 5 | SignalR / Real-Time | 📅 Deferred | Design Phase 10, impl Phase 12 | Half day design |
| 6 | PWA / Offline-First | 📅 Deferred indefinitely | Only if real users demand it | — |
| 7 | Vector DB / RAG | 📅 Deferred | Decide before Phase 17 design | Half day decision |
| 8 | OpenTelemetry | 📅 Deferred | Phase 8 (with Dapr) | One day |
| 9 | Tenant Onboarding / Offboarding | 📅 Deferred | Design before Phase 12 | Half day design |
| 10 | CORS security fix | 🔄 Do now | Immediate (30 min) | 30 min |
| 10b | SOC 2 path | 📅 Deferred | Design before Phase 14 | Half day design |
| 11 | Feature Flags | 📅 Deferred | Before Phase 14 | One day |
| 12 | Durable Workflows | ✅ Resolved via Dapr Workflow | Phase 13 | Covered by Gap 1 |
| 13 | OpenAPI → TypeScript Client Generation | 📅 Deferred | Before Phase 12 | Half day |
| 14 | External Integration Resilience Layer | 📅 Deferred | Before Phase 9 | Half day |
| 15 | Tenant-Configurable Rules Engine | 📅 Deferred | Design before Phase 10 | One day |
| 16 | Polly / Resilience Pipelines | 📅 Deferred | Before Phase 9 | 2 hours |
| 17 | Caching Strategy | ✅ Resolved | Phase 5 (ICacheService) + Phase 8 (Dapr eval) | — |
| 18 | Output Caching for Read Endpoints | 📅 Deferred | Phase 8 | 1 hour |
| 19 | Hangfire / Job Observability | 📅 Deferred | Before Phase 11 | Half day |
| 20 | Structured Health Checks | 📅 Deferred | Before Phase 16 | 2 hours |
| 21 | Idempotency Keys | 📅 Deferred | Design before Phase 12 | Half day |
| 22 | Infrastructure as Code | 📅 Deferred — options open | Phase 16 | One day |
| 23 | .NET 8 → .NET 10 Upgrade | 📅 Deferred | Phase 16 | Half day |
| 24 | .NET Aspire + Dapr Combination | 📅 Deferred | Evaluate Phase 8 | Half day |
| 25 | FastEndpoints for Integration/AI Modules | 📅 Deferred | Phase 14 design / Phase 17 impl | Half day per module |

---

## Gap Details

---

### Gap 1 — Dapr ✅ Phase 8 Candidate

**Decision:** Dapr is the **Phase 8 candidate** to replace hand-built distributed infrastructure. NOT a current-phase concern. NOT locked — decision confirmed at Phase 8 when multi-pod concerns are real.

**What Dapr would replace (if adopted at Phase 8):**

| Phase | Current plan | With Dapr |
|---|---|---|
| 8 | `DistributedCacheService` (Redis direct) | `DaprCacheService` (State building block) |
| 9 | OutboxMessage + BackgroundService retry | Dapr pub/sub + `[Topic]` subscribers |
| 13 | Work Permit approvals via BackgroundService | Dapr Workflow (durable, survives pod restarts) |
| 15 | Rewrite outbox for Azure Service Bus | YAML component swap — zero code change |
| 16 | Wire Key Vault SDK directly | Dapr Secrets building block |

**Why not now (Phase 5):** No multi-pod concern yet. Adopting Dapr before Phase 8 adds sidecar complexity without any of the benefits (cross-pod state, pub/sub resilience, durable workflows). The `ICacheService` abstraction wired in EHS-51 makes the Phase 8 swap a single DI line change.

**Why the ICacheService abstraction matters:**
```
Phase 5 (EHS-51):    DistributedCacheService : ICacheService  ← Redis via IDistributedCache
Phase 8 evaluation:  DaprCacheService : ICacheService         ← Dapr State building block
Swap cost:           One DI line change in DependencyInjection.cs. Zero handler changes.
```

**Dapr local dev command (when adopted at Phase 8):**
```
dapr run --app-id ehs-api --app-port 5147 --components-path ./dapr/components -- dotnet run
```

**NuGet (Phase 8, not now):** `Dapr.Client` (Infrastructure), `Dapr.AspNetCore` (API).

**Career note:** Dapr is CNCF-backed (same family as Kubernetes, Prometheus). Appears in .NET, Go, Java, Python job descriptions. Broader than just .NET — genuine cloud-native portfolio value.

---

### Gap 2 — NetArchTest 🔄 DO NOW (half day)

**What:** Automated tests that enforce Clean Architecture layer rules. If Domain ever imports Infrastructure, the build fails — not a code review catch.

**Why now:** Before Phase 6 adds more files and before any migration to complex phases. The cost of fixing a layer violation scales with codebase size. A half-day now prevents a painful refactor later.

**Example tests:**
```csharp
[Fact]
public void Domain_Should_Not_Reference_Infrastructure()
{
    Types.InAssembly(DomainAssembly)
        .ShouldNot().HaveDependencyOn("EHSPlatform.Infrastructure")
        .GetResult().IsSuccessful.Should().BeTrue();
}

[Fact]
public void Application_Should_Not_Reference_API()
{
    Types.InAssembly(ApplicationAssembly)
        .ShouldNot().HaveDependencyOn("EHSPlatform.API")
        .GetResult().IsSuccessful.Should().BeTrue();
}
```

**NuGet:** `NetArchTest.Rules`. New test project: `EHSPlatform.ArchTests`.

**Effort:** Half a day. No production code changes. Zero risk.

**When:** Standalone ticket before Phase 6 starts. High value, low risk.

---

### Gap 3 — API Versioning 📅 Before Phase 12

**What:** URL path versioning (`/api/v1/`) via `Asp.Versioning.Mvc` NuGet.

**Why before Phase 12:** Once the React frontend locks onto the API contract, breaking changes require a new version. Without versioning wired in advance, every API change is a breaking change.

**Options:**
- URL path: `/api/v1/`, `/api/v2/` — most visible, simplest to test (recommended)
- Header: `Api-Version: 1.0` — cleaner URLs
- Query string: `?api-version=1.0` — easy to test from browser

**Recommendation:** URL path via `Asp.Versioning.Mvc`. Wire once, all future endpoints get it.

**Effort:** One day. Set up before Phase 12 React work starts.

---

### Gap 4 — DB Migration Strategy for Production 📅 Before Phase 16

**Problem:** Locally, `dotnet ef database update` is fine. In production with live tenant data:
- A migration could lock a large table for minutes
- Failed halfway — how do you roll back?
- Multiple pods starting simultaneously could each try to migrate

**Decision:** EF Migration Bundles with a manual approval gate in the ACA GitHub Actions pipeline. Never auto-migrate in production.

```yaml
# GitHub Actions pipeline (Phase 16)
- name: Apply migrations (manual approval required)
  run: ./efbundle --connection ${{ secrets.PROD_CONNECTION_STRING }}
  environment: production  # requires manual approval in GitHub
```

**Effort:** One day to design and wire. Non-negotiable before any production deployment.

---

### Gap 5 — SignalR / Real-Time 📅 Design Phase 10, Implement Phase 12

**Features requiring real-time:**
- Emergency muster — "Is everyone accounted for?" updates as workers scan in
- Active work permit register — who is working where, live
- Dashboard counts — incident/near-miss totals without F5

**Architecture fit:** SignalR hooks into existing MediatR domain events. Add `IRealtimeNotificationService` interface in Application, implement via `IHubContext<MusterHub>` in Infrastructure. Existing handlers don't change.

**ACA multi-pod concern:** Azure SignalR Service as a backplane (one DI line: `.AddAzureSignalR(connectionString)`). Deferred until multi-pod scaling is actually needed in production.

**When:** Design the site check-in data model at Phase 10 (Gap — who is on site). Implement SignalR at Phase 12 when emergency muster is built. Azure SignalR Service only when scaling beyond one pod.

---

### Gap 6 — PWA / Offline-First 📅 DEFERRED INDEFINITELY

**Decision:** Only if real field users with real signal problems explicitly demand it.

**If built:** A separate standalone deliverable — dedicated React PWA project or React Native app. The backend API never changes. Clean Architecture means only the client layer changes.

**What exists as a partial substitute:** Work Permit QR scanning (Phase 13) will include offline queuing for scan events — the highest-urgency offline scenario.

---

### Gap 7 — Vector DB / RAG 📅 Decide Before Phase 17 Design

**When vector DB is needed:** Semantic search across thousands of historical incidents at scale. Context window limits when sending incidents to AI API. "Find incidents similar to this one from 3 years ago" at high volume.

**Simple approach works first:** Typed DTOs → Claude API covers moderate data volumes (single tenant, hundreds of incidents). No vector DB needed until scale demands it.

**For structured document extraction (insurance PDFs, certificates):**
- **Temperature = 0** — fully deterministic extraction. Same PDF → same output, every time
- **Strict JSON schema prompt** — constrain output format, eliminating variation
- **Document hash cache** — SHA-256 hash of PDF bytes; extract once, cache result, never re-extract the same file

```
PDF uploaded → SHA-256 hash → already in cache? → return cached JSON (no AI call)
If new → Claude API (Temperature=0, JSON schema) → store against hash → return
```

**Options when vector DB is needed:**
- **pgvector** — SQL Server vector extension (or switch to Postgres), cheapest, no new infra
- **Azure AI Search** — managed, integrates with Azure OpenAI, higher cost
- **Qdrant** — open-source, self-hostable, excellent .NET SDK

**Connects to:** EHS-58 (Insurance tracking), Phase 10 document workflows.

---

### Gap 8 — OpenTelemetry 📅 Phase 8

**What's missing:** Metrics (request rates, error rates, queue depths) and distributed traces (the full journey of one request).

**Logs** are already covered by Serilog ✅.

**Good news:** If Dapr is adopted at Phase 8, it auto-exports OTEL traces for every sidecar call (state, pub/sub, secrets) — zero extra code. Adding `OpenTelemetry.Extensions.Hosting` + metrics instrumentation for HTTP, EF Core, and custom business metrics (incidents per hour, active tenants) covers the remainder.

**Note:** `.NET Aspire` (Gap 24) wires OpenTelemetry automatically as part of service defaults — Gaps 8, 16, and 20 resolved for free if Aspire is adopted at Phase 8.

---

### Gap 9 — Tenant Onboarding / Offboarding 📅 Design Before Phase 12

**What's missing:** Automated flow for signup (new tenant created, default roles provisioned, admin invited, trial started) and GDPR-compliant offboarding (data exported, tenant soft-deleted, billing stopped).

**Currently:** Tenants created manually via `POST /api/auth/register`. Acceptable for Phase 5, not for Phase 12 multi-tenant UI.

**Decisions needed before Phase 12:**
- Self-service signup or admin-provisioned only?
- Trial period: days-based, feature-limited, or both?
- GDPR data export format: JSON, CSV?
- How long to retain soft-deleted tenant data?
- Default rule set provisioned at tenant creation (connects to Gap 15 Rules Engine)

---

### Gap 10 — CORS Security Fix 🔄 DO NOW (30 minutes)

**Problem:** The current API almost certainly uses `AllowAnyOrigin` for local dev convenience. Before any deployment — even to a test environment — this must be locked down.

**Current (wrong for production):**
```csharp
builder.Services.AddCors(o => o.AddPolicy("AllowAll", p => p.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader()));
```

**Correct:**
```csharp
builder.Services.AddCors(o => o.AddPolicy("EHSPolicy", p =>
    p.WithOrigins(configuration["AllowedOrigins"]!.Split(','))  // from appsettings
     .AllowCredentials()
     .AllowAnyMethod()
     .AllowAnyHeader()));
```

**appsettings.Development.json:** `"AllowedOrigins": "http://localhost:5173"` (Vite dev server)
**appsettings.json:** `"AllowedOrigins": "https://app.ehsplatform.com"`

**When:** Next Phase 5 ticket or as a standalone 30-minute fix. No excuse to defer this.

---

### Gap 10b — SOC 2 Path 📅 Design Before Phase 14

**Foundation already built:** Azure SQL Ledger Tables ✅, soft deletes ✅, RowVersion ✅, CorrelationId ✅.

**Remaining:** Define what constitutes a "security event" for `SecurityAuditLog`, document data retention policies, plan annual pen test, classify data sensitivity per entity.

**When:** Design before Phase 14 when enterprise clients become a target. Not a code problem — it's audit trails, access logs, pen testing evidence, and operational procedures.

---

### Gap 11 — Feature Flags 📅 Before Phase 14

**Why it matters:** Already plan tier gating (Free/Standard/Professional/Enterprise for AI features). Feature flags extend this — beta-test a new form with 10 tenants, disable a buggy feature instantly, gradually roll out Phase 12 React UI.

**Recommendation:** Azure App Configuration — integrates with ACA, handles targeting by tenant, supports percentage rollouts.

**Alternative:** Custom `FeatureFlags` table in the DB — simpler, sufficient for Phase 8-13.

**When:** Design before Phase 14. MediatR pipeline behaviours (`AuditBehaviour`, `LoggingBehaviour`) are already modular — they can be conditionally skipped per tenant flag. No over-engineering needed now.

---

### Gap 12 — Durable Workflows ✅ Resolved via Dapr (Gap 1)

**Problem solved:** Work Permit multi-step approval chains (Phase 13) need to survive pod restarts, wait hours for an approver, and escalate on no-response. `BackgroundService` cannot do any of these reliably.

**Decision:** Dapr Workflow handles this natively at Phase 13. The Rules Engine (Gap 15) is the gate; Dapr Workflow is the approval chain after the gate passes.

```
Worker submits Work Permit
    ↓
[WorkflowRulesBehaviour — preconditions evaluated]
    InsuranceValid? ✅   TrainingNotExpired? ✅
    ↓ (if all pass)
[Dapr Workflow starts]
    Gas Tester notified → waits up to 2hrs → Supervisor confirms → Permit Active
```

---

### Gap 13 — OpenAPI → TypeScript Client Generation 📅 Before Phase 12

**What:** Auto-generate the React frontend's HTTP client from the Swagger spec. Backend changes an endpoint → TypeScript compiler flags every broken frontend call before runtime.

**Recommendation:** NSwag — integrates into the .NET build pipeline, generates idiomatic TypeScript, no extra tooling.

**Alternative:** Kiota (Microsoft's newer generator), openapi-typescript-codegen (npm).

**Effort:** Half a day. Set up before Phase 12 React work starts. Frontend type safety from day one.

---

### Gap 14 — External Integration Resilience Layer 📅 Before Phase 9

**Problem:** No standard for calling external APIs (badge systems, insurance verifiers, email gateways). Each call would be ad-hoc — no retry, no circuit breaker, no timeout, no CorrelationId forwarding.

**Solution:** Composed `DelegatingHandler` pipeline + Polly `AddStandardResilienceHandler` registered once per `HttpClient`:

```csharp
builder.Services.AddHttpClient<BadgeVerificationService>()
    .AddHttpMessageHandler<CorrelationIdForwardingHandler>()
    .AddHttpMessageHandler<OutboundLoggingHandler>()
    .AddStandardResilienceHandler(); // retry + circuit breaker + timeout (Polly)
```

**Per-tenant integration config (when tenants use different providers):**
```
TenantIntegrationConfig: TenantId | IntegrationType | BaseUrl | ApiKey | IsEnabled
```

**Fail policy per integration:** Hard block (pharma/nuclear), fail-open (construction), or cached-rights (use last known answer up to TTL).

**Effort:** Half a day to establish the pattern. Near-zero per new integration after that.

---

### Gap 15 — Tenant-Configurable Rules Engine 📅 Design Before Phase 10

**What:** A `WorkflowRule` table where each tenant configures their own preconditions. Rules are evaluated via a MediatR pipeline behaviour — handlers never know the rules exist.

**Example rules:**
```
ClientA: WorkPermitApplication requires InsuranceValid + TrainingNotExpired → Block if fails
ClientB: BadgeIssuance requires SecurityProviderCheck → Block if fails
ClientC: WorkPermitApplication requires OkToWorkScoreAbove80 → Block if fails
```

**MediatR pipeline behaviour:**
```csharp
public class WorkflowRulesBehaviour<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(TRequest request, HandlerDelegate next, CancellationToken ct)
    {
        if (request is IHasWorkflowTrigger triggered)
        {
            var rules = await _ruleRepo.GetForTenant(triggered.TenantId, triggered.Trigger, ct);
            var failures = await _evaluator.EvaluateAsync(rules, triggered.Context, ct);
            if (failures.Any(f => f.OnFail == RuleAction.Block))
                throw new WorkflowPreconditionException(failures);
        }
        return await next();
    }
}
```

**Connects to:** EHS-56 ("Ok to Work" score is the output of evaluating all rules), Gap 12 (Dapr Workflow is what runs after the gate passes), Gap 11 (Feature Flags for tenant admin UI).

**Recommendation:** Start with a custom condition evaluator (condition types are finite and known). Add Microsoft Rules Engine if conditions become complex or client-authored.

**Effort:** Schema + evaluator + MediatR behaviour = one day. Tenant admin UI = Phase 14.

---

### Gap 16 — Polly / Resilience Pipelines 📅 Before Phase 9

**Problem:** Polly is completely absent. Every external call in Phase 9+ (email, blob storage, badge providers) is a direct call with no retry, no circuit breaker, no timeout. One SendGrid outage fails the entire command.

**Also:** The hand-rolled `RetryCount` + `Error` fields on `OutboxMessage` are a custom retry system. Polly replaces that logic with one pipeline definition.

```csharp
builder.Services.AddHttpClient<IEmailService, SendGridEmailService>()
    .AddStandardResilienceHandler(); // Polly default: 3 retries, exponential backoff, 10s timeout
```

**Circuit breaker behaviour:** After 3 failures, Polly stops calling the failing service for 30 seconds — returns fast failure instead of hanging per concurrent user.

**Effort:** 2 hours to add NuGet + `.AddStandardResilienceHandler()` to all `HttpClient` registrations. Near-zero per new integration.

---

### Gap 17 — Caching Strategy ✅ Resolved

**Decision (May 2026):** Two-phase approach using `ICacheService` abstraction for clean swap path.

| Phase | Tool | Notes |
|---|---|---|
| Phase 5 (EHS-51) | `DistributedCacheService : ICacheService` backed by `IDistributedCache` (StackExchange.Redis) | Infrastructure plumbing. No caching logic yet. |
| Phase 7+ | Activate `ICacheService` in query handlers | Reference data, org settings, aggregate counts |
| Phase 8 evaluation | Optionally swap to `DaprCacheService : ICacheService` (Dapr State building block) | One DI line change if Dapr adopted |

**FusionCache:** Rejected. Adds L1 in-memory + stampede protection but the abstraction overhead is not justified at current scale. `ICacheService` + Dapr State achieves equivalent benefits at Phase 8 without the intermediate dependency.

**What NOT to cache (critical rule):** Never cache live operational state (Incidents, Permits, CorrectiveActions) with a TTL. Cache only: reference data, lookup tables, org settings, user permission maps, aggregate counts.

**Cache key convention:** `tenant:{tenantId}:{entity}:{id}`

---

### Gap 18 — Output Caching for Read Endpoints 📅 Phase 8

**.NET 8 built-in** HTTP response caching — eliminates manual cache-key management for read-heavy reference data endpoints.

```csharp
[OutputCache(Duration = 300, VaryByRouteValueNames = new[] { "siteId" })]
[HttpGet("/api/sites/{siteId}/areas")]
public async Task<IActionResult> GetSiteAreas(Guid siteId) { ... }
```

The entire HTTP response is cached for 300 seconds, varied by `siteId`. No handler changes. No cache keys. Tag-based invalidation available.

**Applies to:** Site areas, organisation structure, user permissions, contractor types, induction groups. Not for incident data or anything that changes per user action.

**Effort:** One attribute per endpoint. 1 hour total across all reference data endpoints.

---

### Gap 19 — Hangfire / Job Observability 📅 Before Phase 11

**Problem:** By Phase 11, multiple background jobs run continuously. `BackgroundService + PeriodicTimer` gives no dashboard, no retry history, no dead-letter queue. Silent failures go undetected until a client reports missed notifications.

**Hangfire adds (free tier):**
- Web dashboard at `/jobs` — job history, success/failure rates, duration
- Automatic retry with configurable backoff
- Dead-letter queue for jobs failing N times
- Cron scheduling (replaces `PeriodicTimer`)
- **SQL Server storage** — already in your stack, no extra infrastructure

**Note:** Dapr pub/sub (Gap 1) handles event-driven messaging. Hangfire handles scheduled recurring jobs. They are complementary, not competing.

**Effort:** Half a day. `BackgroundService` acceptable for Phase 9 MVP. Must be in before job count exceeds 2-3.

---

### Gap 20 — Structured Health Checks 📅 Before Phase 16

**.NET 8 built-in:**
```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>("database")
    .AddRedis(redisConnectionString, "redis")
    .AddCheck<SendGridHealthCheck>("email-provider", tags: new[] { "external" });

app.MapHealthChecks("/health/live");  // is process alive?
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = reg => !reg.Tags.Contains("external") // don't block readiness on external APIs
});
```

ACA wires `/health/live` and `/health/ready` to liveness/readiness probes in `containerapp.yaml` — zero app code change after initial setup.

**Effort:** 2 hours. Must be in before production.

---

### Gap 21 — Idempotency Keys 📅 Design Before Phase 12

**Problem:** Network drops mid-POST → client retries → two identical incidents created. In an EHS compliance system, duplicates corrupt audit trails.

**Solution:** `Idempotency-Key` header + MediatR pipeline behaviour:
```csharp
POST /api/incidents
Headers: Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

// IdempotencyBehaviour checks IDistributedCache before handler runs
// If key found → return previous response (don't re-execute)
// If new → run handler, cache result with 24hr TTL
```

**Which commands need it:** `CreateIncidentCommand`, `SubmitWorkPermitCommand`, `CreateCorrectiveActionCommand`.

**Storage:** `IDistributedCache` (Redis). Keys expire after 24 hours.

**Effort:** Half a day design + implementation. Pattern registered once in MediatR pipeline.

---

### Gap 22 — Infrastructure as Code 📅 Phase 16 — Options Open

**No final decision yet.** Options documented for selection based on cost, client requirements, or hosting choice at Phase 16.

| Tool | Best when |
|---|---|
| **Bicep** | Azure-only deployments, tightest Azure integration, same-day new feature support |
| **Terraform** | Multi-cloud, appears on every enterprise JD, client may require it |
| **Pulumi** | Want to write infrastructure in C# alongside the app |
| **Helm** | If hosting moves to AKS instead of ACA (client Kubernetes requirement) |

**Current hosting preference:** ACA — native Dapr support, scales to zero, no Kubernetes ops overhead. If client requires Kubernetes → AKS + Helm is the upgrade path.

**What needs to be provisioned (Phase 16):** Resource Group, ACA Environment, Azure SQL, Azure Cache for Redis, Azure Service Bus, Key Vault, Container Registry, Azure SignalR Service (when Gap 5 activated).

---

### Gap 23 — .NET 8 → .NET 10 Upgrade 📅 Phase 16

**Skip .NET 9** (STS — already near end of life by Phase 16). Go 8 → 10 directly.

| Version | Type | Support Until |
|---|---|---|
| .NET 8 (current) | LTS | November 2026 |
| .NET 9 | STS | May 2026 — **skip** |
| .NET 10 | LTS | November 2028 — upgrade to this |

**How clean:** Change `<TargetFramework>` in all `.csproj` files. Domain and Application: zero changes (pure C#, no framework deps). Infrastructure and API: minor breaking changes to fix. Run tests, done.

**Effort:** Half a day as a dedicated Phase 16 step before ACA deployment.

---

### Gap 24 — .NET Aspire + Dapr Combination 📅 Evaluate Phase 8

**What they are:**

| | Dapr | .NET Aspire |
|---|---|---|
| Layer | Runtime (runs in production) | Dev-time orchestration + service defaults |
| What it does | State, pub/sub, workflow, secrets | Orchestrates local startup, pre-wires OTEL + Polly + health checks |
| In production? | Yes — sidecar alongside app | No — dev orchestration only |

**They work together (officially supported):**
```csharp
// Aspire AppHost/Program.cs
var api = builder.AddProject<Projects.EHSPlatform_API>("ehs-api")
    .WithDaprSidecar(); // starts Dapr sidecar automatically
```

**Aspire solves Gaps 8, 16, and 20 automatically** — OpenTelemetry, Polly resilience, and structured health checks all come for free from Aspire's service defaults. One command starts: API + Dapr sidecar + Redis + SQL Server + Aspire dashboard.

**When:** Evaluate at Phase 8 when Dapr is being evaluated. Full adoption at Phase 16 when multiple services (API, AI, notification worker, React frontend) need orchestrating locally.

---

### Gap 25 — FastEndpoints for Specific Modules 📅 Phase 14 Design / Phase 17 Impl

**NOT a wholesale migration.** MVC controllers + MediatR stays for all core business logic.

**Where FastEndpoints genuinely belongs:**

1. **Integration API project (Phase 14)** — badge reader check-in events (high frequency, every worker scan), security provider webhook callbacks, IoT sensor data from hazardous areas, public developer API for plugin architecture. These don't need MediatR pipeline overhead. FastEndpoints gives ~3× throughput over MVC for these patterns.

2. **AI Inference Endpoints (Phase 17)** — streaming responses, `IAsyncEnumerable` — clean and fast without MediatR.

3. **Minimal APIs (built-in .NET, no NuGet)** — webhook signature validation, file upload progress endpoints, internal health probes.

**Resume value:** Demonstrating a deliberate architectural decision — "main business logic uses Clean Architecture + MediatR for correctness; high-throughput integration surface uses FastEndpoints for performance" — is senior-level thinking.

---

## Competitive Analysis — EHASoft SHEQ Features (May 2026)

Audit conducted of EHASoft SHEQ Contractor Portal (maiworks.net/cp3). Features not yet planned in our 17-phase roadmap.

### HIGH PRIORITY — Roadmap Impact

| Feature | Gap | Proposed Ticket | Target Phase |
|---|---|---|---|
| Expiry Notification Engine | No expiry tracking or automated notification system | EHS-56 | Phase 9 |
| "Ok to Work" Composite Score | No compliance health indicator per contractor/worker | EHS-57 | Phase 10 |
| Document Review Workflow (RAMS) | No general document review workflow — RAMS is a legal requirement in UK/EU construction | EHS-58 | Phase 10 |
| Insurance Tracking | Not planned anywhere in roadmap | EHS-59 | Phase 10 |
| Scheduled Automated Reports | No automated reporting planned | EHS-63 | Phase 11 |

### MEDIUM PRIORITY — Worth Planning

| Feature | Gap | Proposed Ticket | Target Phase |
|---|---|---|---|
| Induction Management (full lifecycle) | Scheduling, slots, contractor type → group mapping not planned | EHS-61 | Phase 10 |
| Contractor Vendor Rating (Bronze/Silver/Gold) | Not planned | EHS-60 | Phase 10 |
| Site Check-In Data Model | Foundation for emergency muster — who is on site, when | EHS-62 | Phase 10/12 |

### LOWER PRIORITY / STRATEGIC

| Feature | Gap | Notes |
|---|---|---|
| Plugin / Integration Architecture | Not planned | Design before Phase 14 — connects to Gap 25 |
| Azure AD / Entra ID Provisioning | Deferred | Phase 14 SSO (already in technical-debt.md) |
| Multilingual UI (7 languages) | Not planned | Required for EU enterprise (chemical, energy) — design before Phase 12 |
| AI Search | SHEQ already ships it | Add semantic search Phase 12 using existing AI infrastructure (Groq cheap) |

### Our Advantages Over EHASoft SHEQ

| Our Feature | Why It Differentiates |
|---|---|
| Full Incident Reporting + CAPA cycle | Full accountability loop — not just compliance tracking |
| Work Permit (Permit to Work) | Legal requirement in high-risk industries — SHEQ contractor portal doesn't show this |
| Emergency Muster (Phase 12) | Active safety headcount, not just an access log |
| AI Pattern Detection (Phase 17) | "This contractor type has 3× incident rate at Site A" |
| Semantic Form Engine (Phase 7) | AI-readable adaptive forms — industry first in EHS SaaS |
| Just Culture language enforcement | Encourages reporting, reduces blame — measurable safety culture impact |
| Near Miss tracking | Leading indicator — SHEQ tracks documents, not near misses |
| Dapr-backed pub/sub resilience | More reliable than their apparent jQuery + server-rendered stack |

---

## Proposed New Tickets (from Competitive Analysis + Gap Review)

These tickets have been identified but not yet created in Jira. Parth to review and create when appropriate.

| Ticket | Description | Phase | Source |
|---|---|---|---|
| EHS-55 | ETag/If-Match HTTP contract — RowVersion visible to API clients (created May 2026) | Phase 5 | technical-debt.md #13 |
| EHS-56 | Expiry date + expiry notification pub/sub event on Document, Training, Insurance entities | Phase 9 | Competitive analysis |
| EHS-57 | "Ok to Work" composite score — computed per contractor/employee | Phase 10 | Competitive analysis |
| EHS-58 | RAMS document type + multi-stage document review workflow | Phase 10 | Competitive analysis |
| EHS-59 | Insurance tracking fields on ClientContractor entity | Phase 10 | Competitive analysis |
| EHS-60 | Contractor vendor rating (Bronze/Silver/Gold + risk level + valid until) | Phase 10 | Competitive analysis |
| EHS-61 | Induction scheduling — site/day/time slots, contractor type → group mapping | Phase 10 | Competitive analysis |
| EHS-62 | Site check-in data model — foundation for emergency muster and Security Desk | Phase 10/12 | Competitive analysis |
| EHS-63 | Scheduled automated reports — configurable frequency to named recipients | Phase 11 | Competitive analysis |

> **Note:** EHS-55 (ETag/If-Match) was created in Jira in May 2026. EHS-56 through EHS-63 are proposed tickets from the competitive analysis — not yet created in Jira. Create when the relevant phase begins.
