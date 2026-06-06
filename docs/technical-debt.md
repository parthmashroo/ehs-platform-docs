# EHS Platform — Technical Debt & Architecture Review Log

> Living document. Updated after every post-commit architecture review.
> Each entry: what was found, why it matters, target phase, ticket.

---

## How to Read This

| Severity | Meaning |
|---|---|
| 🔴 High | Genuine design flaw — causes bugs or data corruption if not fixed |
| 🟡 Medium | Architectural smell — creates tech debt, harder to fix later |
| 🟢 Low | Hardening / nice-to-have — safe to defer, no functional risk |

---

## Phase 2 Review — Incident Lifecycle (Apr 2026)

### 🔴 Domain state is not structurally protected
**Finding:** `Incident.Status` and `Incident.AssignedToId` have public setters. Any handler can bypass `TransitionTo()` and set status directly — `incident.Status = IncidentStatus.Closed`. The state machine protection is a convention, not enforced by the compiler.

**Risk:** A future developer adds a handler and sets status directly. State machine silently bypassed. In a compliance domain this is a genuine correctness issue.

**Fix:** Make `Status` and `AssignedToId` private setters. Add `Incident.AssignTo(Guid userId)` domain method that both sets the assignee and auto-transitions — replacing the logic currently in `AssignIncidentHandler`.

**Target phase:** Phase 3
**Ticket:** EHS-32
**Status:** ⬜ Open

---

### 🔴 Auto-transition business rule lives in the handler, not the domain
**Finding:** `AssignIncidentHandler` auto-transitions to `UnderInvestigation` when assigning. But `UpdateIncidentCommand` also allows setting `AssignedToId` without triggering this rule. Same business rule, two code paths — only one enforces it.

**Fix:** Move to `Incident.AssignTo()` domain method. Covered by the same fix as above.

**Target phase:** Phase 3
**Ticket:** EHS-32
**Status:** ⬜ Open

---

### ~~🟡 Pagination is unstable — same record can appear on two pages~~ ✅ Fixed in EHS-33
**Finding:** `OrderByDescending(x => x.OccurredAt)` with no tiebreaker. If two incidents share the same `OccurredAt` (common in bulk imports or rapid reporting), SQL Server returns them in non-deterministic order. Page 1 and page 2 can return the same record.

**Fix:** `.OrderByDescending(x => x.OccurredAt).ThenBy(x => x.Id)` — Id as tiebreaker gives deterministic ordering.

**Target phase:** Phase 3
**Ticket:** EHS-33
**Status:** ⬜ Open

---

### ~~🟡 No upper bound on pageSize — memory bomb risk~~ ✅ Fixed in EHS-33
**Finding:** `GET /api/incidents?pageSize=100000` is valid. One authenticated call loads every incident in the tenant into memory. At 10k incidents × 100 tenants, this is a DoS vector from inside the app.

**Fix:** `var safePageSize = Math.Min(request.PageSize, 100);` in the handler. Two lines.

**Target phase:** Phase 3
**Ticket:** EHS-33
**Status:** ⬜ Open

---

### 🔴 No optimistic concurrency — concurrent edits cause silent data loss
**Finding:** No `RowVersion` / concurrency token on `Incident`. Two users editing the same incident simultaneously — last write wins silently. No 409 Conflict. No indication to either caller that their write may have overwritten the other's.

**Fix:** Add `byte[] RowVersion` to `Incident` (or `BaseEntity`). Configure `IsRowVersion()` in EF. Map `DbUpdateConcurrencyException` → 409 in `ExceptionHandlingMiddleware`.

**Target phase:** Phase 4
**Ticket:** EHS-34
**Status:** ⬜ Open

---

### 🟡 AssignedToId and ReportedById reference non-existent users
**Finding:** Both are bare `Guid` fields with no FK constraint to a User table (User entity doesn't exist until Phase 4). Any Guid is accepted — including invalid ones. Test data uses Organization Guid as User Guid.

**Fix:** Add FK constraint to User table when Phase 4 (Auth) lands. Add validator rule to check existence once `IApplicationDbContext.Users` exists.

**Target phase:** Phase 4
**Ticket:** EHS-35
**Status:** ⬜ Open

---

### 🟢 PATCH /assign returns 204 — caller blind to status change
**Finding:** `AssignIncident` auto-transitions status, but returns 204 No Content. Caller has no idea the status changed and must do a separate GET to find out. Hidden side effect in the API contract.

**Fix:** Return 200 with the updated incident status, or at minimum the new `status` field in the response body.

**Target phase:** Phase 4 or when frontend starts (Phase 12)
**Ticket:** EHS-36
**Status:** ⬜ Open

---

### 🟡 UpdateCorrectiveAction allows edits on terminal states
**Finding:** `UpdateCorrectiveActionCommandHandler` updates fields (Title, Description, Priority, AssignedToId, DueDate) with no check on current status. A `Cancelled` or `Verified` corrective action should be immutable — these are closed states. Editing them silently undermines the audit trail.

**Risk:** A caller can change the title of a Verified CA after it has been signed off. In a compliance context this is a data integrity issue — the verified record no longer reflects what was actually verified.

**Fix:** Add a guard in the handler before applying field updates:
```csharp
if (ca.Status is CorrectiveActionStatus.Verified or CorrectiveActionStatus.Cancelled)
    throw new InvalidOperationException("Cannot edit a corrective action in a terminal state.");
```

**Target phase:** Phase 3
**Ticket:** EHS-37
**Status:** ⬜ Open

---

### 🟢 Re-open from Closed is unrestricted
**Finding:** `TransitionTo()` allows any non-Reported state → Reported (re-open), including `Closed`. In a compliance context, closing an incident is often a formal/legal act. Re-opening from Closed with no role restriction is dangerous once auth is in place.

**Fix:** When Phase 4 auth lands, add a role guard: only `SafetyOfficer` or `OrganizationAdmin` can re-open a Closed incident. The domain method should accept an optional `UserRole` parameter or use a domain service.

**Target phase:** Phase 4
**Ticket:** EHS-35 (include with user entity work)
**Status:** ⬜ Open

---

## Deferred Architectural Decisions

These are not bugs or debt — they are deliberate deferrals. The design today is intentionally simple. These items should be revisited before acquiring enterprise clients or onboarding tenants with divergent compliance workflows.

---

### 🔵 Configurable workflows per tenant — hardcoded state machines won't scale to enterprise SaaS
**Context:** Every entity with a state machine (Incident, CorrectiveAction, future: Audit, CAPA, Permit) currently has a fixed enum-based status lifecycle. In a real multi-tenant enterprise SaaS, different clients have different workflows — some require a Verified step, some don't; some have 3 statuses, some have 8.

**The real problem:** As we add more clients, "can you configure the workflow?" will be one of the first enterprise questions. Right now the answer is no.

**Options when this comes up:**
- **Workflow templates** (Phase 6-8): Define 2-3 fixed workflows per module (Simple / Standard / Strict), tenant picks at onboarding. Covers 80% of clients.
- **Database-driven workflows** (Phase 10+): Statuses and transitions stored per tenant in DB. Full flexibility. Complex.
- **Workflow engine** (Phase 14+): External engine (Elsa, Temporal). Full enterprise-grade. High investment.

**Decision today:** Fixed state machines. Design domain methods (`TransitionTo()`) so the transition rules can be injected or overridden later without a full rewrite.

**Target phase:** Phase 10 (multi-tenant configuration module)
**Status:** 🔵 Deferred by design

---

### 🔵 Three specific code locations that break when custom workflows land

**Context:** When Phase 10 introduces tenant-configurable statuses, these three hardcoded patterns across the codebase will need to change:

1. **`AnyAsync` checks in handlers — hardcoded "what counts as done"**
   ```csharp
   x.Status != CorrectiveActionStatus.Completed && x.Status != CorrectiveActionStatus.Verified
   ```
   This hardcodes two enum values as terminal. A tenant with a custom `"Peer Reviewed"` status that also counts as done will silently fail this check. Every handler with this pattern needs updating.

2. **Guards in `TransitionTo()` — hardcoded which transition triggers cross-entity checks**
   ```csharp
   if (newStatus == IncidentStatus.Resolved && hasOpenCorrectiveActions)
   ```
   Hardcodes `Resolved` as the status that triggers the CA check. If a tenant renames or replaces this status, the guard never fires.

3. **Switch expressions — hardcoded transition pairs**
   ```csharp
   (IncidentStatus.AwaitingAction, IncidentStatus.Resolved) => true
   ```
   Fixed enum pairs. Custom statuses live in DB, not enums — the switch breaks entirely.

**Why this is not a problem today:** The `bool hasOpenCorrectiveActions` parameter is the seam. In Phase 10, the same boolean gets populated from a cached tenant config query instead of hardcoded enum values. The domain method signature never changes — only the one line in the handler that populates the boolean.

**Performance concern and answer:** Adding workflow config queries per request would add latency. The solution is Redis caching (Phase 8) — workflow config is read-heavy, write-rare (changed maybe monthly per tenant). One cache miss per config change, then ~0.5ms Redis hits for every subsequent request. The trade: 0.5ms latency for zero-deployment tenant configuration. Correct trade for a SaaS.

**Phase 10 data model:**
```
WorkflowStatus    — per tenant, per module: name, CountsAsDone (bool), IsFinal (bool)
WorkflowTransition — per tenant: FromStatusId, ToStatusId, RequiresCrossEntityCheck (bool)
WorkflowCrossEntityRule — per tenant: which linked module must be in terminal status
```

**Target phase:** Phase 10
**Status:** 🔵 Deferred by design

---

### 🔵 One user can belong to multiple tenants with different roles — current model doesn't support this

**Context:** Phase 4 builds `User` with a single `TenantId` and single `Role`. In the real world a safety consultant or contractor supervisor works across multiple companies — Sarah is SafetyOfficer at SafetyCorp AND ContractorAdmin at OilCorp. With current model she needs two accounts. Broken UX.

**Correct long-term model — junction table:**
```
User                      UserTenantMembership
────                      ────────────────────
Id                        UserId   → User
Email                     TenantId → Organisation
PasswordHash              Role
                          IsActive
```
One user record, many tenant+role memberships. At login, if Sarah has multiple memberships she picks which context to enter. Her JWT is stamped with the chosen tenantId + role. Switching context = new JWT, no re-login required.

**Impact on Phase 4:** Build `User` entity knowing role moves to the junction table in Phase 5. No hard design decisions that block the migration.

**Target phase:** Phase 5 (alongside multi-tenancy work)
**Status:** 🔵 Deferred by design

---

### 🔵 Multiple SSO providers — Microsoft, Google, Okta via OIDC standard

**Context:** Phase 4 uses email + password (passwords in our DB). Enterprise clients will demand SSO. Different clients use different providers — large enterprises use Azure AD, US mid-market uses Okta, tech companies use Google.

**The key insight:** Microsoft, Google, Okta, GitHub, Apple all speak the same standard — **OpenID Connect (OIDC)**. We learn it once, every provider works. After verification the code is identical regardless of provider — all reduce to: `(email, providerTenantId) → our tenantId + role → our JWT`.

**DB design per tenant:**
```
ssoProvider:      "microsoft" | "google" | "okta" | "none"
providerTenantId: their Microsoft tid / Google domain / Okta domain
```

**Tenant identification:** In Phase 4 we use email domain or explicit registration. In Phase 14 we use the provider's `tid` claim — exact, unambiguous, doesn't depend on email format.

**Multi-company users (Sarah in two companies):** She picks her context at login. JWT is stamped with chosen tenantId + role. Switching context = new JWT.

**Target phase:** Phase 14
**Status:** 🔵 Deferred by design

---

### 🟢 All detail endpoints embed child lists — system-wide pattern should be count + lazy load
**Finding:** `GetIncidentByIdQueryHandler` embeds the full `CorrectiveActions` list in the incident detail response. This pattern, if repeated across all modules, means opening any detail view loads all child records the user may never scroll to. At scale (incident with 20 CAs, audit with 30 findings) this adds unnecessary payload and DB load.

**Correct pattern — consistent across all modules:**
- Detail response includes child **count only** (e.g. `"correctiveActionCount": 3`)
- Full child list loaded lazily via existing filtered list endpoint when user expands the section
- Example: `GET /api/correctiveactions?incidentId={id}` — already built, called on expand

**Why this matters for UX/consistency:**
Every module (Incidents, Audits, Permits, CAPA) will have child relationships. If each detail screen lazy-loads its children via the same pattern, the UI/UX is consistent across the entire product — collapse/expand everywhere, fast initial load everywhere. Users learn the pattern once.

**Fix (Phase 12):** When frontend starts, update all `GetXxxByIdQueryHandler` responses to return count fields instead of embedded lists. The child list endpoints are already built and ready.

**Target phase:** Phase 12 (when frontend is built — no point fixing before UI exists)
**Status:** ⬜ Open

---

### 🟡 SoftDelete logic lives in the handler, not the domain
**Finding:** `SoftDeleteCorrectiveActionCommandHandler` sets `ca.IsDeleted = true` and `ca.UpdatedAt` directly — bypassing the domain layer. Every other mutation on `CorrectiveAction` goes through `TransitionTo()`. This inconsistency means when EHS-37 lands (reason required when deleting a Verified CA), the guard will likely be added to the handler instead of the domain — repeating the exact mistake EHS-32 caught on `Incident`.

**Fix:** Add `SoftDelete(string? reason = null)` domain method to `CorrectiveAction`:
```csharp
public void SoftDelete(string? reason = null)
{
    IsDeleted = true;
    UpdatedAt = DateTime.UtcNow;
}
```
Handler calls `ca.SoftDelete(request.Reason)` instead of setting fields directly. When EHS-37 guard is added, it goes inside this method — not the handler.

**Target phase:** Phase 3
**Ticket:** EHS-38
**Status:** ⬜ Open

---

### 🔴 User-facing error messages contain developer language and wrong HTTP status codes
**Finding:** Four categories of problems across all exception messages returned in API responses:

1. **`InvalidStatusTransitionException` hardcodes "incident" — but is also thrown for Corrective Actions.** Message template: `"Cannot transition incident from '{from}' to '{to}'"`. A caller cancelling a corrective action receives: *"Cannot transition incident from 'Open' to 'Cancelled'."* — factually wrong entity name.

2. **Raw enum names exposed as status labels.** All messages use `.ToString()` on C# enums, producing PascalCase developer identifiers: `UnderInvestigation`, `AwaitingAction`, `InProgress`. A safety officer reads: *"Cannot transition incident from 'AwaitingAction' to 'Reported'."*

3. **`NotFoundException` exposes entity class names and raw GUIDs.** Template: `"{entityName} with ID '{key}' was not found."` Called with `nameof(CorrectiveAction)`, `nameof(Incident)`. Returns: *"CorrectiveAction with ID '3fa85f64-...' was not found."* to end users.

4. **`InvalidOperationException` in `UpdateCorrectiveActionCommandHandler` returns HTTP 500.** The terminal state guard throws `InvalidOperationException`, which is not handled in `ExceptionHandlingMiddleware` — falls through to `_ => InternalServerError`. A business rule violation returns 500 instead of 422. Message also uses jargon: *"Cannot edit a corrective action in a terminal state."*

**Affected locations:**
- `EHSPlatform.Domain/Exceptions/InvalidStatusTransitionException.cs` — "incident" hardcoded in message template
- `EHSPlatform.Domain/Exceptions/NotFoundException.cs` — exposes class names and raw GUIDs
- `EHSPlatform.Domain/Entities/Incident.cs` — 3 throws using raw `.ToString()` on enum values
- `EHSPlatform.Domain/Entities/CorrectiveAction.cs` — 2 throws using raw `.ToString()` on enum values
- `EHSPlatform.Application/CorrectiveActions/Commands/UpdateCorrectiveAction/UpdateCorrectiveActionCommandHandler.cs` — wrong exception type (500 not 422) + jargon message

**Fix (when sprint is scheduled):**
- Introduce `DomainValidationException` in `EHSPlatform.Domain.Exceptions` — middleware maps it to 422 Unprocessable Entity
- Fix `InvalidStatusTransitionException` to accept entity name as a constructor parameter or make template entity-agnostic
- Add `ToDisplayName()` extension method on all status enums — maps enum values to plain English display strings used in all messages
- Rewrite `NotFoundException` template: *"The requested [resource] could not be found."*
- Rewrite all guard reason strings to plain language a safety officer would understand

**Target phase:** Dedicated error message sprint — schedule when backlog reaches ~5 user-language issues
**Ticket:** EHS-44
**Status:** ⬜ Open

---

## Phase 5 Architectural Gap Review (May 2026)

### 🔴 CORS not locked to specific origins — security exposure before any deployment

**Finding:** The API currently uses `AllowAnyOrigin` (or no CORS policy at all) for local dev convenience. Before any deployment — even to a test environment — this must be locked to specific allowed origins. Any browser-based attacker on any domain can make authenticated API calls from a user's browser.

**Fix:**
```csharp
// Program.cs — replace AllowAll policy
builder.Services.AddCors(o => o.AddPolicy("EHSPolicy", p =>
    p.WithOrigins(builder.Configuration["AllowedOrigins"]!.Split(','))
     .AllowCredentials().AllowAnyMethod().AllowAnyHeader()));

// appsettings.Development.json
"AllowedOrigins": "http://localhost:5173"

// appsettings.json (production)
"AllowedOrigins": "https://app.ehsplatform.com"
```

**Risk:** Without this, any malicious site can make CORS requests to the API using a logged-in user's JWT stored in the browser. Simple attack, catastrophic in a compliance system.

**Effort:** 30 minutes. No excuse to defer.

**Target phase:** Immediately. Add to EHS-51 sprint or as a standalone 30-minute task.
**Ticket:** Needs creation. Add to Phase 5 backlog.
**Status:** ⬜ Open

---

### 🟡 No architecture layer tests — Clean Architecture enforced by discipline only

**Finding:** The rule "Domain must not reference Infrastructure" is enforced by code review alone. A developer at 2am can add a bad import and it will compile cleanly. By the time it's caught in review, other code may depend on it.

**Fix:** Add `EHSPlatform.ArchTests` test project with `NetArchTest.Rules` NuGet. Three assertions cover all layer rules:
```csharp
Domain     → must not reference Application, Infrastructure, or API
Application → must not reference Infrastructure or API
Infrastructure → must not reference API
```

**Risk:** Without this, Clean Architecture is a convention, not a constraint. Violations compound silently.

**Effort:** Half a day. No production code changes. Zero regression risk. Pure read-only assertions.

**Target phase:** Phase 5 or 6 — before the codebase grows larger.
**Ticket:** EHS-61
**Status:** ⬜ Open

---

## Phase 5 Review — EHS-46 RowVersion (May 2026)

### 🟡 ETag/If-Match HTTP contract missing — RowVersion invisible to clients
**Finding:** `RowVersion` is wired at the DB layer (SQL Server `timestamp` column, `DbUpdateConcurrencyException` → 409), but the HTTP API contract is incomplete. `GET` endpoints don't return an `ETag` response header, and `PUT`/`PATCH` endpoints don't check an `If-Match` request header. A real client has no way to read the current `RowVersion` or send it back on a write — the 409 can never be intentionally triggered by a legitimate caller.

**Risk:** The concurrency protection exists in the DB but is invisible and unreachable through the API. Any client doing a read-then-write will never get a 409 — it will silently overwrite because it has no way to supply the current `RowVersion`. The DB-level protection only fires if two clients somehow send the exact same stale token, which can't happen without the API contract.

**Fix:** For every mutable endpoint (PUT/PATCH on Incidents, CorrectiveActions):
1. `GET /api/incidents/{id}` → include `ETag: "<base64-rowversion>"` response header
2. `PUT /api/incidents/{id}` → read `If-Match` header, populate `RowVersion` on the loaded entity before `SaveChangesAsync`. If header absent → 428 Precondition Required. If header mismatches → 409 Conflict.

**Why deferred from EHS-46:** The DB-side work (RowVersion column + 409 handler) ships independently and gives partial protection against internal race conditions. The HTTP contract layer is a separate concern that touches every controller and query handler — better scoped as its own ticket.

**Target phase:** Phase 5 (alongside remaining EHS-5x tickets)
**Ticket:** EHS-55 (created May 2026)
**Status:** ⬜ Open

---

## Phase 6 Review — EHS-56 AuditLog Entity (May 2026)

### 🔴 AuditLog.ChangedAt is DateTime — must be DateTimeOffset before EHS-57
**Finding:** `AuditLog.ChangedAt` is typed as `DateTime`. In a multi-region SaaS (EU-hosted or otherwise), `DateTime` without an embedded offset is ambiguous — app servers in different timezones produce inconsistent timestamps. Querying or sorting across instances can miss records or produce wrong ordering. The correct type is `DateTimeOffset`, which stores the UTC value plus the offset. The rule: always store UTC, let the UI handle locale display (timezone, currency, date format per user preference).

**Fix:** In `AuditLog.cs`, change `public DateTime ChangedAt { get; set; }` to `public DateTimeOffset ChangedAt { get; set; }`. Run a new migration to update the column type. Also update `AuditLogEntryDto` in EHS-58 to use `DateTimeOffset`. Must be done before EHS-57 starts — EHS-57 writes to this field.

**Target phase:** Phase 6 — fix before EHS-57
**Ticket:** Fold into EHS-57 pre-work
**Status:** ⬜ Open

---

### 🔴 No GDPR right-to-erasure strategy — ChangedById FK blocks User deletion
**Finding:** `AuditLog.ChangedById` uses `DeleteBehavior.Restrict` — correct for audit integrity, but means a User record can never be deleted while any AuditLog row references them. Under GDPR Article 17 (right to erasure), a user in an EU-regulated jurisdiction can request deletion of their personal data. The `ChangedById` Guid (and potentially PII embedded in `OldValues`/`NewValues` JSON blobs) may constitute personal data under GDPR.

**Recommended fix — Anonymisation:** On erasure request, replace `ChangedById` with a sentinel `WellKnownGuids.DeletedUser` Guid and redact PII from JSON blobs. The audit trail is preserved for compliance; user identity is removed for GDPR. Preferable to full row deletion, which destroys the compliance record.

**Why not full deletion:** An EHS SaaS is itself a compliance platform. Deleting audit rows to satisfy GDPR creates a paradox — you destroy the evidence that safety procedures were followed. Anonymisation satisfies GDPR while preserving the compliance record.

**Target phase:** Phase 9+ (compliance/legal module)
**Ticket:** EHS-60
**Status:** ⬜ Open

---

## Phase 6 Review — EHS-57 AuditInterceptor (May 2026)

### 🟡 AuditLog missing ChangedByName — user display name not stored at write time

**Finding:** `AuditLog.ChangedById` stores a Guid. The display name (e.g. "John Smith") is resolved at read time via a DB join in EHS-58. This is the ServiceNow pattern — their most complained-about audit limitation. If a user is hard-deleted (GDPR erasure), the Guid becomes unresolvable and the audit entry loses its "who" context permanently.

**Industry standard:** SAP, Dynamics 365, EHASoft all store the display name at write time — immutable historical truth. The name captured at event time is what the record showed at that moment, even if the person later changes their name or is removed.

**Fix:** Add `ChangedByName string` column to `AuditLog`. Populate from a `name` JWT claim in the interceptor. Requires adding `name` claim to `LoginCommandHandler` and exposing `UserName` on `ICurrentUserService`. Migration needed.

**Why deferred:** `ICurrentUserService` does not expose a name today — only UserId, TenantId, Role, CompanyType. Adding it requires a JWT claim change (Phase 8 work). EHS-58 resolves names via DB join as an acceptable interim.

**Target phase:** Phase 8
**Ticket:** Fold into Phase 8 JWT / token enrichment work
**Status:** ⬜ Open

---

### 🟡 AuditInterceptor blind to bulk operations and raw SQL — silent audit gaps

**Finding:** The interceptor hooks into EF Core's `ChangeTracker`. Any operation that bypasses the ChangeTracker leaves zero audit trail:
- `context.Incidents.Where(...).ExecuteUpdateAsync(...)` — EF 7+ bulk update, no ChangeTracker
- `context.Database.ExecuteSqlRawAsync(...)` — raw SQL, no ChangeTracker
- Direct DBA changes, migration scripts, data fixes

Currently not a risk — all handlers load entities before modifying. But as the platform grows (bulk close, bulk archive, data migrations), this gap will silently appear.

**Fix:** SQL Server Temporal Tables as a complementary DB-level safety net (see item below). Application AuditLog captures intent (who, why, correlation). Temporal Tables capture everything the DB touches — including bulk ops and DBA changes.

**Target phase:** Phase 16 (pre-production hardening)
**Ticket:** Needs creation
**Status:** ⬜ Open

---

### 🟡 Disconnected entity updates produce empty audit diffs

**Finding:** If an entity is attached to the context without being loaded from DB (`context.Entry(entity).State = EntityState.Modified`), EF Core has no original values snapshot. `OriginalValue` equals `CurrentValue` for all properties. The interceptor writes an AuditLog row where `OldValues` and `NewValues` are identical — technically not wrong, practically useless.

**Currently not a risk:** All handlers load the entity from DB before modifying. The pattern `var entity = await _context.Incidents.FindAsync(id)` is used everywhere — EF takes its snapshot on that load. Guard this as a code review rule: never use `context.Entry().State = Modified` on an entity that wasn't loaded from the current context.

**Target phase:** Enforce as convention. Revisit if bulk-import patterns are introduced.
**Status:** ⬜ Monitor

---

### 🟢 AuditLogs table has no partitioning strategy — grows unbounded

**Finding:** The shared `AuditLogs` table has a composite index `(TenantId, EntityName, EntityId)` but no partitioning. At 10M+ rows, index fragmentation accumulates, maintenance jobs (rebuild/reorganise) become expensive, and query performance degrades for cross-entity or time-range scans.

**Fix:** SQL Server table partitioning by `ChangedAt` (monthly or yearly buckets). Old partitions can be moved to cheaper storage or archived. Zero application code change — DML operations are partition-aware automatically.

**When it matters:** At realistic EHS platform scale (500 sites, 10,000 incidents/year, all changes audited), 10M rows takes roughly 10–20 years. Monitor actual row counts at Phase 16 before acting.

**Target phase:** Phase 16 (pre-production, based on actual data volume)
**Ticket:** Needs creation
**Status:** ⬜ Open

---

### 🟢 SQL Server Temporal Tables not implemented — no DB-level tamper-proof history

**Finding:** The application AuditLog captures intent (who, why, correlation, semantic labels). It does not capture full row snapshots and is limited to ChangeTracker-visible operations. SQL Server Temporal Tables provide a complementary DB-level guarantee:
- Full row snapshot on every write — tamper-proof, maintained by SQL Server engine
- Point-in-time reconstruction: `SELECT * FROM Incidents FOR SYSTEM_TIME AS OF '2026-01-01'`
- Works for bulk ops, raw SQL, DBA changes — everything that touches the DB
- Zero application code — one migration per table

**What temporal tables don't capture (that our AuditLog does):** ChangedById, CorrelationId, AuditAction label, intent/reason. Both layers together give the complete picture: what changed (temporal tables) + who and why (AuditLog).

**Compliance implication:** ISO 45001 and OSHA auditors expect legible, tamper-evident change history. Temporal tables are the strongest technical proof of record integrity available in SQL Server without a third-party system.

**Target phase:** Phase 16 (pre-production compliance hardening)
**Ticket:** Needs creation
**Status:** ⬜ Open

---

### 🟢 AuditLog UI — EHS-58 must produce field-level diff structure, not raw JSON blobs

**Finding:** Industry standard EHS audit UI (EHASoft, Intelex, Cority) presents a `Field | Old Value | New Value` grid with changed rows visually highlighted. Raw JSON blobs stored in `OldValues`/`NewValues` are not directly displayable. EHS-58 query response must parse the JSON and produce a structured DTO that the UI can render as a grid without interpretation.

**EHS-58 response shape:**
```json
{
  "changedAt": "...",
  "changedBy": "John Smith",
  "action": "Updated",
  "changes": [
    { "field": "Status", "oldValue": "Reported", "newValue": "Under Investigation" },
    { "field": "AssignedTo", "oldValue": null, "newValue": "Sarah Jones" }
  ]
}
```

Enum values are already stored as strings (e.g. `"Reported"`) — no translation needed at read time. User names require a DB join until `ChangedByName` is added (see item above). Field name → display label mapping is EHS-58's responsibility.

**Target phase:** EHS-58 (next ticket in Phase 6)
**Status:** ⬜ Design decision captured — implement in EHS-58

---

## Phase 6 Review — EHS-58 Audit Log Endpoints (May 2026)

### 🟢 `a.Action.ToString()` inside LINQ select — client-side evaluation risk
**Finding:** The audit log handler calls `a.Action.ToString()` inside the LINQ `select` projection. EF Core may not be able to translate this enum `.ToString()` call to SQL and silently switch to client-side evaluation — fetching the full result set and evaluating in memory.

**Fix:** Project `Action = (int)a.Action` and call `.ToString()` after `.ToListAsync()`, or store a local `var action = a.Action;` before the query and reference that inside the select.

**Target phase:** Phase 6/7
**Status:** ⬜ Open

---

### 🟢 `IgnoreQueryFilters()` on Users join — undocumented scope
**Finding:** Both audit log handlers use `_context.Users.IgnoreQueryFilters()` in the join to access soft-deleted users. `IgnoreQueryFilters()` bypasses ALL query filters on `Users` — not just `IsDeleted`. If a TenantId filter is ever added to `Users`, this silently becomes a cross-tenant user name leak.

**Fix:** Add a comment at every `IgnoreQueryFilters()` call explaining which filter is being bypassed and why. When a TenantId filter is added to Users, revisit all call sites.

**Target phase:** Phase 8 (when Users gets TenantId filter)
**Status:** ⬜ Open

---

### 🟡 `TenantId == Guid.Empty` guard missing in audit log handlers
**Finding:** Both `GetIncidentAuditLogQueryHandler` and `GetCorrectiveActionAuditLogQueryHandler` use `_currentUser.TenantId` as the tenant filter. If `TenantId == Guid.Empty` (authenticated but no tenant — edge case tracked in existing debt), the handler silently returns an empty list rather than failing fast with a clear error.

**Fix:** Add guard at the top of both handlers:
```csharp
if (_currentUser.TenantId == Guid.Empty)
    return [];
```

**Target phase:** Phase 6
**Status:** ⬜ Open

---

## Phase 7 Review — EHS-69 CORS (Jun 2026)

### 🟡 CORS integration test relies on Development environment config — fragile in CI

**Finding:** `CorsIntegrationTests` sends a preflight for `http://localhost:3000` and expects `Access-Control-Allow-Origin` in the response. The allowed origin exists only in `appsettings.Development.json`. `WebApplicationFactory` loads this file because the test environment defaults to `Development`. If any CI environment sets `ASPNETCORE_ENVIRONMENT=Production` (or any non-Development value), the `Cors:AllowedOrigins` array is empty from the base `appsettings.json` and both CORS tests break.

**Fix:** In the `CorsIntegrationTests` constructor, inject origins via `builder.ConfigureAppConfiguration(cfg => cfg.AddInMemoryCollection(new[] { new KeyValuePair<string, string?>("Cors:AllowedOrigins:0", "http://localhost:3000") }))`. This makes the test self-contained and environment-agnostic.

**Target phase:** Phase 7
**Status:** ⬜ Open

---

### 🟢 CORS `WithHeaders` allow-list has no maintenance process — new custom headers will silently fail

**Finding:** `AddCors` explicitly allows `Content-Type`, `Authorization`, `X-Correlation-Id`. Any new custom request header added to the API (e.g. `X-Idempotency-Key`, `X-Api-Version`) will be blocked by the browser with a CORS error until it is added to this list. There is no automated guard to catch the omission.

**Fix:** Document the allow-list in `CLAUDE.md` so that any new custom header addition triggers a CORS allow-list update. Optionally, add an architecture test that asserts the allow-list matches a known enumerable.

**Target phase:** Ongoing
**Status:** ⬜ Open

---

## Summary Table

| # | Finding | Severity | Target | Ticket | Status |
|---|---|---|---|---|---|
| 1 | Public status/assignee setters — state machine bypassable | 🔴 High | Phase 3 | EHS-32 | ✅ Fixed |
| 2 | Auto-transition rule in handler, not domain | 🔴 High | Phase 3 | EHS-32 | ✅ Fixed |
| 3 | Unstable pagination — no sort tiebreaker | 🟡 Medium | Phase 3 | EHS-33 | ✅ Fixed |
| 4 | No pageSize cap — memory bomb | 🟡 Medium | Phase 3 | EHS-33 | ✅ Fixed |
| 5 | No optimistic concurrency — silent data loss | 🔴 High | Phase 4 | EHS-34/46 | ✅ Fixed |
| 6 | AssignedToId/ReportedById — no FK or existence check | 🟡 Medium | Phase 4 | EHS-35 | ✅ Fixed |
| 7 | PATCH /assign returns 204 — status change invisible | 🟢 Low | Phase 4/12 | EHS-36 | ✅ Fixed |
| 8 | Re-open from Closed unrestricted | 🟢 Low | Phase 4 | EHS-35 | ✅ Fixed |
| 9 | UpdateCorrectiveAction edits terminal states — audit trail risk | 🟡 Medium | Phase 3 | EHS-37 | ✅ Fixed |
| 10 | SoftDelete logic in handler, not domain — inconsistent, no seam for future guard | 🟡 Medium | Phase 3 | EHS-38 | ✅ Fixed |
| 11 | All detail endpoints embed child lists — system-wide pattern should be count + lazy load | 🟢 Low | Phase 12 | — | ⬜ Open |
| 12 | User-facing error messages: developer language + wrong HTTP status codes | 🔴 High | Error sprint | EHS-44 | ✅ Fixed |
| 13 | ETag/If-Match HTTP contract missing — RowVersion invisible to clients | 🟡 Medium | Phase 5 | EHS-55 | ⬜ Open |
| 14 | CORS not locked — AllowAnyOrigin before any deployment | 🔴 High | Phase 7 | EHS-69 | ✅ Fixed |
| 15 | No architecture layer tests — Clean Architecture enforced by discipline only | 🟡 Medium | Phase 5/6 | EHS-61 | ⬜ Open |
| 16 | All timestamps are DateTime — must be DateTimeOffset globally (BaseEntity + all entities + all handlers) | 🔴 High | Phase 6 (pre-EHS-57) | EHS-62 | ✅ Fixed |
| 17 | No GDPR right-to-erasure strategy — ChangedById FK blocks User deletion | 🔴 High | Phase 9+ | EHS-60 | ⬜ Open |
| 18 | AuditLog missing ChangedByName — display name not stored at write time | 🟡 Medium | Phase 8 | — | ⬜ Open |
| 19 | AuditInterceptor blind to bulk ops and raw SQL — silent audit gaps | 🟡 Medium | Phase 16 | — | ⬜ Open |
| 20 | Disconnected entity updates produce empty audit diffs | 🟡 Medium | Convention | — | ⬜ Monitor |
| 21 | AuditLogs table — no partitioning strategy at scale | 🟢 Low | Phase 16 | — | ⬜ Open |
| 22 | SQL Server Temporal Tables not implemented — no DB-level tamper-proof history | 🟢 Low | Phase 16 | — | ⬜ Open |
| 23 | EHS-58 returns raw JSON blobs for OldValues/NewValues — field-level diff DTO deferred | 🟡 Medium | Phase 12 | — | ⬜ Deferred |
| 24 | Incoming DateTimeOffset values from clients not normalized to UTC — client can submit +05:30 offset and it persists as-is | 🟡 Medium | Phase 7 (before API goes public) | — | ✅ Fixed |
| 25 | `a.Action.ToString()` inside LINQ select — may force client-side evaluation against SQL Server | 🟢 Low | Phase 6/7 | — | ⬜ Open |
| 26 | `IgnoreQueryFilters()` on Users join bypasses ALL future filters — undocumented scope risk | 🟢 Low | Phase 8 (when Users gets TenantId filter) | — | ⬜ Open |
| 27 | `TenantId == Guid.Empty` guard missing in audit log handlers — silent empty result for unauthenticated tenant | 🟡 Medium | Phase 6 | — | ⬜ Open |
| 28 | CORS integration test relies on `appsettings.Development.json` — fails silently if env is not Development | 🟡 Medium | Phase 7 | EHS-69 | ✅ Fixed |
| 29 | CORS `WithHeaders` allow-list has no maintenance process — new custom headers will cause silent browser CORS errors | 🟢 Low | Ongoing | — | ⬜ Open |
