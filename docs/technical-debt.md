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

### ~~🟢 `a.Action.ToString()` inside LINQ select — client-side evaluation risk~~ ✅ Fixed in EHS-70
**Finding:** The audit log handler calls `a.Action.ToString()` inside the LINQ `select` projection. EF Core may not be able to translate this enum `.ToString()` call to SQL and silently switch to client-side evaluation — fetching the full result set and evaluating in memory.

**Fix:** Project `ActionInt = (int)a.Action` in the SQL expression tree, call `((AuditAction)r.ActionInt).ToString()` in-memory after `.ToListAsync()`. Applied to both `GetIncidentAuditLogQueryHandler` and `GetCorrectiveActionAuditLogQueryHandler`.

**Target phase:** Phase 6/7
**Status:** ✅ Fixed — EHS-70

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

## Phase 7 Multi-Reviewer Audit (Jun 2026)

> Whole-codebase review by 4 independent agents (architecture, security, database, code-quality). Findings below are where 2+ reviewers converged or a single reviewer found a high-impact issue. Convergence noted per entry — independent agreement = high confidence the issue is real.

### 🔴 MediatR ValidationBehavior missing — every FluentValidation validator is dead code
**Finding:** `AddValidatorsFromAssembly` registers all validators, but no `IPipelineBehavior` invokes them and there is no MVC auto-validation. Commands dispatch via `ISender.Send`, so MVC model-validation never runs either. Every `*CommandValidator` (Create/Update Incident, all CorrectiveAction commands, Login, Register) is unreachable. Invalid input (empty Title, future OccurredAt, non-UTC offset) reaches the DB and only fails on a constraint. `RegisterCommandHandler` hand-throws `ValidationException` as a one-off workaround — proof the pipeline gap was felt but not fixed. **Convergence: Architecture + Code-quality.**

**Fix:** `ValidationBehavior<TRequest,TResponse>` in `Application/Common/Behaviours/`, resolves `IEnumerable<IValidator<TRequest>>`, runs all, throws `ValidationException`. Register `AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>))`. Map to `ValidationProblemDetails` (field-level) in middleware. Add a test asserting invalid command → 400.

**Target phase:** Phase 7 (immediate — highest leverage)
**Ticket:** Needs creation (P0-1)
**Status:** ⬜ Open

---

### 🔴 Tenant isolation depends on per-handler discipline, not an enforced seam
**Finding:** Three independent gaps, same root cause. (a) `User`/`Organization` global query filter omits `TenantId` (`ApplicationDbContext.cs:44-47`) — any Users query returns every tenant's users (PII, emails, roles). `AssignIncidentCommandHandler:24` exploits this: assigns incidents to foreign-tenant users. (b) `TenantId` never stamped on Incident/CorrectiveAction create (`CreateIncidentCommandHandler:26`, `CreateCorrectiveActionCommandHandler:22`) → FK violation (feature broken) or mis-owned rows. (c) No interceptor enforces either; `User` does not implement `ITenantEntity`, which is why the filter gap went uncaught. **Convergence: Architecture + Security + Database (all 3).**

**Fix:** `User` implements `ITenantEntity`; filter `x.TenantId == _tenantId && !x.IsDeleted`. Login/Register pre-auth lookups use explicit `IgnoreQueryFilters()`. `TenantStampInterceptor` (SaveChanges) stamps `TenantId` on `Added` `ITenantEntity` rows. `CreateCorrectiveAction` validates `IncidentId` belongs to current tenant before insert (IDOR fix). `ICurrentUserService` becomes a required DbContext dependency (remove `= null`); write path asserts `TenantId != Guid.Empty`.

**Target phase:** Phase 7 (immediate)
**Ticket:** Needs creation (P0-2) — one epic covering the seam
**Status:** ⬜ Open

---

### 🔴 JWT signing key hardcoded in committed appsettings.json
**Finding:** `"SecretKey": "EHSPlatform-Dev-SecretKey-..."` in base `appsettings.json:8` (HS256 symmetric). Ships to all environments unless overridden. Anyone with the repo forges a JWT with any `tid` + `Role=OrganizationAdmin` → full cross-tenant admin. **Convergence: Security (single reviewer, highest exploit impact).**

**Fix:** Remove from committed config; user-secrets (dev), env/Key Vault (prod). Startup guard throws in non-Development if missing or equal to the known dev value. Pin `ValidAlgorithms = [HmacSha256]`. **Rotate the exposed key — it is compromised (was in git).**

**Target phase:** Phase 7 (immediate)
**Ticket:** Needs creation (P0-3)
**Status:** ⬜ Open

---

### 🔴 SQL Server columns are datetime2, not datetimeoffset — offset silently discarded at DB layer
**Finding:** Distinct from EHS-62 (which fixed the C# property type to `DateTimeOffset`). Every `DateTimeOffset` property maps to SQL `datetime2` across every entity config — no `HasColumnType("datetimeoffset")`. The offset is dropped on write; EF reconstructs `+00:00` on read, hiding the loss locally. `DateTimeOffsetUtcZConverter` is a JSON serialization converter (API only), not an EF value converter — it does not cover the column type. Violates ADR-016 intent at the persistence layer. **Convergence: Database (authoritative single reviewer).**

**Fix:** `HasColumnType("datetimeoffset")` on every `DateTimeOffset` property (CreatedAt, UpdatedAt, OccurredAt, DueDate, CompletedAt, ChangedAt, LastLoginAt). Migration (non-destructive ALTER — wider precision). Do before production data.

**Target phase:** Phase 7 (before prod data)
**Ticket:** Needs creation (P0-4)
**Status:** ⬜ Open

---

### 🔴 AuditLog has no global query filter — tenant isolation hand-rolled per query
**Finding:** `AuditLog` has `TenantId` but no model-level filter; both audit-log handlers manually add `where a.TenantId == tenantId`. One forgotten `WHERE` in a future handler leaks every tenant's audit trail — the most sensitive data in the system. `AuditLog` is append-only (no `IsDeleted`). **Convergence: Architecture + Database.**

**Fix:** `AuditLog` implements `ITenantEntity`; global filter `x.TenantId == _tenantId`. Drop the manual predicates from both handlers. Interceptor already stamps `TenantId` on write.

**Target phase:** Phase 7
**Ticket:** Needs creation (P1, fold into P0-2 seam epic)
**Status:** ⬜ Open

---

### 🟡 exception.Message returned raw on 500 — schema/SQL info disclosure
**Finding:** `ExceptionHandlingMiddleware:52` serializes `exception.Message` on the unmapped `InternalServerError` branch. EF/SQL exceptions leak schema, SQL, connection details. Accelerates the tenant-isolation attacks above. **Convergence: Security.**

**Fix:** 500 branch returns static "An unexpected error occurred"; log the real message server-side only (already logged at :29).

**Target phase:** Phase 7
**Ticket:** Needs creation (P1)
**Status:** ⬜ Open

---

### 🟡 Audit index mismatch + single-column tenant indexes — list/audit queries sort in memory at scale
**Finding:** (a) `AuditLogConfiguration` defines composite indexes including `ChangedAt`, but migration `20260525144047_AddAuditLog` created narrower indexes without it — config/schema drift. Audit queries sort `ChangedAt DESC` in memory. (b) `Incidents`/`CorrectiveActions` `TenantId` indexes are single-column; the primary paginated list queries (filter Status/Severity/SiteId, order OccurredAt/DueDate) degrade to index scan + key lookup + sort as data grows. **Convergence: Database.**

**Fix:** Filtered composites — `IX_Incidents (TenantId, Status, OccurredAt DESC) WHERE IsDeleted=0`, CorrectiveActions equivalent; audit composite includes `ChangedAt`. Align config and migration.

**Target phase:** Phase 7/8
**Ticket:** Needs creation (P1)
**Status:** ⬜ Open

---

### 🟡 No role-based authorization on write/delete — any authenticated user (incl. Contractor) can mutate/delete
**Finding:** Incident/CorrectiveAction controllers guard only with `[Authorize]` (any role). A low-privilege user or `CompanyType=Contractor` can soft-delete corrective actions. The `ctype` trust boundary is never enforced anywhere. **Convergence: Security.**

**Fix:** Per-operation policies (delete → admin/safety officer). Decide and enforce what `Contractor` may do.

**Target phase:** Phase 8
**Ticket:** Needs creation (P1)
**Status:** ⬜ Open

---

### 🟡 Cross-cutting quality debt (claim constants, ctype, audit handler dup, tests)
**Finding (bundle):**
- **Claim-name magic strings** `"uid"/"tid"/"ctype"` duplicated across `JwtTokenService`, `CurrentUserService`, `TenantResolutionMiddleware` — a typo yields a null claim + 500/401. Project's stated #1 priority (magic strings → constants). *(Architecture + Code.)*
- **`CompanyType`/`ctype` claim declared but never consumed** in any authz decision — unwired extensibility seam. The Client/Contractor entitlement differentiation it exists for is unimplemented. *(Architecture.)*
- **Two byte-identical audit-log query handlers** — pure copy-paste incl. `"Unknown User"`/`"(Inactive)"` magic strings. Extract shared builder. *(Database + Code.)*
- **Test coverage shallow** — ~10 test files / ~25 handlers. Zero tests on Assign/Update/Create-CA/all read queries; **no validator unit tests, nothing exercising the validation pipeline** (which is why the dead-validator bug went unnoticed). *(Code.)*

**Fix:** `ClaimNames` constants class. Wire or formally defer `ctype`. Extract shared audit-query builder. Backfill handler + validator tests with the ValidationBehavior work.

**Target phase:** Phase 7/8
**Ticket:** Needs creation (P2 bundle)
**Status:** ⬜ Open

---

### Already tracked elsewhere (confirmed by this audit, not new)
- **RowVersion is inert** — token + 409 mapping exist but no update path passes the original version → silent last-write-wins. Confirmed by Architecture + Database. Already tracked as **EHS-55** (ETag/If-Match HTTP contract, #13 below).
- **Duplicated validation rules** (UTC-offset, future/past date, Title/Desc, Priority) across all 4 Create/Update validators. Confirmed by Architecture + Code. Already covered by the **existing shared-FluentValidation-rules ticket** — append the concrete list: `.MustBeUtc()` extension; Title MaxLength inconsistent (Create 500 vs Update 200); Update validators missing Type/Severity `.IsInEnum()`; Priority `InclusiveBetween(1,4)` on an enum → use `.IsInEnum()`.
- **`TenantId == Guid.Empty` guard missing in audit handlers** — already #27 below.

---

## Phase 7 Review — EHS-65 IAuditableEntity Marker Interface (Jun 2026)

### 🟢 `IAuditableEntity` carries no `Id` contract — interceptor silently breaks if invariant violated

**Finding:** `AuditInterceptor` calls `entry.Property("Id").CurrentValue` on every `IAuditableEntity`. This assumes every auditable entity has an `Id` property — currently guaranteed by `BaseEntity` inheritance. The interface itself enforces nothing. A future entity that implements `IAuditableEntity` without extending `BaseEntity` throws `InvalidOperationException` at runtime with no compile warning.

**Risk:** Low today (all entities use `BaseEntity`). Medium if the team adds a value object or DTO that accidentally implements `IAuditableEntity`.

**Fix:** When EHS-61 architecture tests are written, add an assertion: every type implementing `IAuditableEntity` must also extend `BaseEntity`. Alternatively, move `IAuditableEntity` to a non-marker shape: `public interface IAuditableEntity { Guid Id { get; } }` — but this duplicates the `BaseEntity` declaration and may cause diamond-inheritance noise.

**Target phase:** Phase 7 — fold into EHS-61 arch tests
**Status:** ⬜ Open

---

### 🟢 Generic audit helpers lack `IAuditableEntity` constraint — future extensibility gap

**Finding:** If we add generic audit utilities (e.g. `AuditQueryBuilder<T>`, `GetAuditLogQuery<T>`) in Phase 8+, the type parameter should be constrained to `where T : IAuditableEntity`. Without it, callers can instantiate generic audit helpers against non-auditable types and get runtime failures instead of compile errors.

**Fix:** When the first generic audit helper is written, add `where T : IAuditableEntity` constraint. No action needed today — premature until the helper exists.

**Target phase:** Phase 8+ (when generic audit helpers are introduced)
**Status:** ⬜ Deferred by design

---

### 🟢 No test asserting `AuditLog` does NOT implement `IAuditableEntity`

**Finding:** `AuditLog` is written by the interceptor — it must never be an `IAuditableEntity` itself. If someone adds `IAuditableEntity` to `AuditLog`, the interceptor enters infinite recursion: writing AuditLog triggers another SaveChanges snapshot, triggers another AuditLog write. No guard exists today.

**Fix:** Add to EHS-61 architecture tests: `typeof(AuditLog).IsAssignableTo(typeof(IAuditableEntity))` must be `false`.

**Target phase:** Phase 7 — fold into EHS-61 arch tests
**Status:** ⬜ Open

---

## Phase 7 Review — EHS-66 AuditInterceptor TenantId Guard (Jun 2026)

### 🟢 AuditInterceptor guard fires silently — no observability when malformed auth state hits

**Finding:** `AddAuditLogs()` returns early when `TenantId == Guid.Empty` (EHS-66 guard). The return is silent — no log, no metric. In production, if a middleware bug or malformed JWT causes every `SaveChanges` to skip auditing, there is no signal to diagnose from. The compliance audit trail degrades invisibly.

**Fix:** Inject `ILogger<AuditInterceptor>` and add one warning log on the guard path:
```csharp
if (context is null || !_currentUserService.IsAuthenticated || _currentUserService.TenantId == Guid.Empty)
{
    if (_currentUserService.IsAuthenticated)
        _logger.LogWarning("AuditInterceptor skipped: TenantId is Guid.Empty for authenticated user {UserId}", _currentUserService.UserId);
    return;
}
```
The unauthenticated path (system tasks, migrations) is expected — no warning needed there. The authenticated + empty-tenant path is always a bug — always worth logging.

**Target phase:** Phase 7/8 — fold into P2 quality bundle
**Status:** ⬜ Open

---

## Phase 7 Review — EHS-67 CorrectiveAction Audit Coverage (Jun 2026)

### 🟢 AuditInterceptor uses `GetType().Name` — proxy-unsafe EntityName in production

**Finding:** `AuditInterceptor` derives `EntityName` via `entry.Entity.GetType().Name`. If EF Core lazy-loading proxies are enabled (via `UseLazyLoadingProxies()`), `GetType()` returns the proxy subclass — `"CorrectiveActionProxy"`, `"IncidentProxy"` — not the mapped entity name. Every audit row written under lazy loading stores a wrong `EntityName`, permanently corrupting the audit trail with no error or warning.

**The EHS-67 test does not catch this** — `InMemoryDatabase` does not use proxies. A proxy-enabled integration test against SQL Server would expose it immediately.

**Fix:** Replace `entry.Entity.GetType().Name` with `entry.Metadata.ClrType.Name` in `AuditInterceptor`. The EF Core model metadata always holds the mapped CLR type, not the runtime proxy type. One-character change, zero behavioral difference in non-proxy scenarios.

```csharp
// Before (proxy-unsafe)
var entityId = entry.Property("Id").CurrentValue?.ToString() ?? string.Empty;
// EntityName captured elsewhere as: entry.Entity.GetType().Name

// After — also fix EntityName at the AddRange site:
EntityName = entry.Metadata.ClrType.Name,
```

**Risk:** Low today — `UseLazyLoadingProxies()` is not called anywhere in the project. Medium if any future developer adds it as a "fix" for navigation property N+1 issues. The audit trail corruption would be silent and permanent.

**Target phase:** Phase 7 — small fix, can be done as part of any interceptor touch
**Status:** ⬜ Open

---

## Session 32 Review — Proprietary Vocabulary / ACL Gap (Jun 2026)

### 🟡 Application layer leaks external library API surfaces — no Proprietary Vocabulary seam

**Finding:** 37 files in `EHSPlatform.Application` reference `using MediatR` directly. 15 handler classes implement `IRequestHandler<,>` (MediatR interface). 14 command/query classes implement `IRequest<T>` (MediatR interface). 8+ validators extend `AbstractValidator<T>` (FluentValidation). 10+ entity configs implement `IEntityTypeConfiguration<T>` directly. These library API surfaces leak into EHS's domain layer — the Application layer expresses library vocabulary, not EHS vocabulary.

**The violation:** Application layer should express its own ubiquitous language. `IRequestHandler<CreateIncidentCommand, Guid>` is MediatR vocabulary. `ICommandHandler<CreateIncidentCommand, Guid>` is EHS vocabulary. One tells you a library detail. The other tells you what the class does.

**Compounding cost (the decisive argument):**
- Phase 7 now: 37 files to retrofit + 7 ACL files to create = 2 hours
- Phase 17 if deferred: ~170 files + test refactor + baked muscle memory = 2 days + regression risk
- Every phase that adds handlers without the seam raises the cost. Do it at Phase 7.

**Risk beyond swap protection:** MediatR v13 went commercial July 2025. A CVE in v12 requiring upgrade would currently touch 37 files. With the ACL seam: swap cost drops to 5 files.

**Fix (EHS-81):** 7 ACL interface/base-class files in `Application/Common/CQRS/` and `Application/Common/Validation/`. All handlers, commands, validators, and configs reference EHS interfaces only. MediatR confined to 5 files total after this change. Full detail in ADR-018 (`docs/architecture-decisions.md`).

**Target phase:** Phase 7 — before more handlers are added
**Ticket:** EHS-81
**Status:** ⬜ Open

---

## Phase 7 Review — EHS-72 ValidationBehavior (Jun 2026)

### 🟢 ValidationBehavior runs for all MediatR requests — ICommand marker interface would scope it

**Finding:** `ValidationBehavior<TRequest, TResponse>` is registered as a open-generic `IPipelineBehavior<,>`, so it fires for every MediatR dispatch — including read queries (`GetIncidentsQuery`, `GetAuditLogsQuery`). Queries have no validators; DI resolves an empty `IEnumerable<IValidator<TRequest>>` and the behavior short-circuits. Low cost per call, but architecturally noisy: the behavior is a command-only concern applied system-wide.

**Fix:** Add `ICommand` marker interface. Constrain behavior: `where TRequest : ICommand`. All command classes implement `ICommand`. Queries are untouched. One line per command class; no behavior change.

**Target phase:** Phase 8
**Ticket:** Fold into quality bundle (P2)
**Status:** ⬜ Open

---

### 🟢 Task.WhenAll parallelizes validators — safe today, fragile when async DB rules are added

**Finding:** `ValidationBehavior` runs all validators via `Task.WhenAll(validators.Select(v => v.ValidateAsync(...)))`. Today all validators are pure in-memory (no DB calls). Safe. The moment any validator adds `MustAsync` (e.g., uniqueness checks against the DB), two validators on the same command fire two concurrent DB queries with no rate-limiting, connection-pool awareness, or ordering guarantee.

**Fix:** No action now. When the first `MustAsync` validator is written, revisit: either `foreach` + `await` (sequential), or set a concurrency policy. Document the parallel assumption in the behavior.

**Target phase:** Phase 8+ (when async validators introduced)
**Ticket:** —
**Status:** ⬜ Deferred by design

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
| 25 | `a.Action.ToString()` inside LINQ select — may force client-side evaluation against SQL Server | 🟢 Low | Phase 6/7 | EHS-70 | ✅ Fixed |
| 26 | `IgnoreQueryFilters()` on Users join bypasses ALL future filters — undocumented scope risk | 🟢 Low | Phase 8 (when Users gets TenantId filter) | — | ⬜ Open |
| 27 | `TenantId == Guid.Empty` guard missing in audit log handlers — silent empty result for unauthenticated tenant | 🟡 Medium | Phase 6 | — | ⬜ Open |
| 28 | CORS integration test relies on `appsettings.Development.json` — fails silently if env is not Development | 🟡 Medium | Phase 7 | EHS-69 | ✅ Fixed |
| 29 | CORS `WithHeaders` allow-list has no maintenance process — new custom headers will cause silent browser CORS errors | 🟢 Low | Ongoing | — | ⬜ Open |
| 30 | MediatR ValidationBehavior missing — all FluentValidation validators are dead code | 🔴 High | Phase 7 | EHS-72 | ✅ Fixed |
| 31 | Tenant isolation not an enforced seam — User filter omits TenantId, TenantId not stamped on create, no interceptor | 🔴 High | Phase 7 | P0-2 | ⬜ Open |
| 32 | JWT signing key hardcoded in committed appsettings.json — cross-tenant token forgery | 🔴 High | Phase 7 | P0-3 | ⬜ Open |
| 33 | SQL columns datetime2 not datetimeoffset — offset discarded at DB layer (distinct from EHS-62) | 🔴 High | Phase 7 | P0-4 | ⬜ Open |
| 34 | AuditLog has no global query filter — tenant isolation hand-rolled per query | 🔴 High | Phase 7 | P1 (fold P0-2) | ⬜ Open |
| 35 | exception.Message returned raw on 500 — schema/SQL info disclosure | 🟡 Medium | Phase 7 | P1 | ⬜ Open |
| 36 | Audit index mismatch + single-column tenant indexes — in-memory sorts at scale | 🟡 Medium | Phase 7/8 | P1 | ⬜ Open |
| 37 | No role-based authz on write/delete — Contractor can soft-delete; ctype unenforced | 🟡 Medium | Phase 8 | P1 | ⬜ Open |
| 38 | Quality bundle — claim-name constants, ctype unwired, duplicate audit handlers, shallow tests | 🟡 Medium | Phase 7/8 | P2 | ⬜ Open |
| 39 | `IAuditableEntity` has no Id contract — interceptor assumes BaseEntity inheritance; no test guards AuditLog from accidentally implementing the interface | 🟢 Low | Phase 7 | EHS-61 arch tests | ⬜ Open |
| 40 | Generic audit helpers (future) lack `where T : IAuditableEntity` constraint — compile-time safety gap when helpers are introduced | 🟢 Low | Phase 8+ | — | ⬜ Deferred |
| 41 | AuditInterceptor TenantId guard fires silently — no log when malformed auth state skips auditing | 🟢 Low | Phase 7/8 | P2 | ⬜ Open |
| 42 | AuditInterceptor uses `GetType().Name` — proxy-unsafe; `entry.Metadata.ClrType.Name` is the correct EF idiom | 🟢 Low | Phase 7 | — | ✅ Fixed |
| 43 | `CommonValidatorRules.MustBeUtc()` has no `DateTimeOffset?` overload — nullable date fields silently skip the UTC check | 🟢 Low | Phase 8 (if nullable dates added) | — | ✅ Fixed |
| 44 | `using EHSPlatform.Application.Common.Validation;` required per-file — no global using means developers can write inline UTC checks without seeing `MustBeUtc()` exists | 🟢 Low | Phase 7/8 | — | ✅ Fixed |
| 45 | `ValidationBehavior` resolves validators for ALL MediatR requests including queries — wasted DI resolution; `ICommand` marker would scope it to commands only | 🟢 Low | Phase 8 | EHS-81 | ⬜ Open |
| 46 | `Task.WhenAll` in ValidationBehavior parallelizes validators — safe today (pure in-memory); first async validator with DB uniqueness check fans out unbounded DB connections per command | 🟢 Low | Phase 8+ (when async validators added) | — | ⬜ Deferred |
| 47 | Application layer vocabulary gap — MediatR `IRequestHandler<,>` in 15 handlers, `IRequest<T>` in 14 commands, `AbstractValidator<T>` in 8+ validators leak library API surfaces into domain core. Compounding cost: 37 files now, ~170 at Phase 17. ACL-First seam (ADR-018) closes this. | 🟡 Medium | Phase 7 | EHS-81 | ⬜ Open |
