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

## Phase 5 Review — EHS-46 RowVersion (May 2026)

### 🟡 ETag/If-Match HTTP contract missing — RowVersion invisible to clients
**Finding:** `RowVersion` is wired at the DB layer (SQL Server `timestamp` column, `DbUpdateConcurrencyException` → 409), but the HTTP API contract is incomplete. `GET` endpoints don't return an `ETag` response header, and `PUT`/`PATCH` endpoints don't check an `If-Match` request header. A real client has no way to read the current `RowVersion` or send it back on a write — the 409 can never be intentionally triggered by a legitimate caller.

**Risk:** The concurrency protection exists in the DB but is invisible and unreachable through the API. Any client doing a read-then-write will never get a 409 — it will silently overwrite because it has no way to supply the current `RowVersion`. The DB-level protection only fires if two clients somehow send the exact same stale token, which can't happen without the API contract.

**Fix:** For every mutable endpoint (PUT/PATCH on Incidents, CorrectiveActions):
1. `GET /api/incidents/{id}` → include `ETag: "<base64-rowversion>"` response header
2. `PUT /api/incidents/{id}` → read `If-Match` header, populate `RowVersion` on the loaded entity before `SaveChangesAsync`. If header absent → 428 Precondition Required. If header mismatches → 409 Conflict.

**Why deferred from EHS-46:** The DB-side work (RowVersion column + 409 handler) ships independently and gives partial protection against internal race conditions. The HTTP contract layer is a separate concern that touches every controller and query handler — better scoped as its own ticket.

**Target phase:** Phase 5 (alongside EHS-48 TenantId work)
**Ticket:** Needs creation — suggest EHS-54
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
| 13 | ETag/If-Match HTTP contract missing — RowVersion invisible to clients | 🟡 Medium | Phase 5 | — | ⬜ Open |
