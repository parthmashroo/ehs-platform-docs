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

### 🟢 Re-open from Closed is unrestricted
**Finding:** `TransitionTo()` allows any non-Reported state → Reported (re-open), including `Closed`. In a compliance context, closing an incident is often a formal/legal act. Re-opening from Closed with no role restriction is dangerous once auth is in place.

**Fix:** When Phase 4 auth lands, add a role guard: only `SafetyOfficer` or `OrganizationAdmin` can re-open a Closed incident. The domain method should accept an optional `UserRole` parameter or use a domain service.

**Target phase:** Phase 4
**Ticket:** EHS-35 (include with user entity work)
**Status:** ⬜ Open

---

## Summary Table

| # | Finding | Severity | Target | Ticket | Status |
|---|---|---|---|---|---|
| 1 | Public status/assignee setters — state machine bypassable | 🔴 High | Phase 3 | EHS-32 | ⬜ Open |
| 2 | Auto-transition rule in handler, not domain | 🔴 High | Phase 3 | EHS-32 | ⬜ Open |
| 3 | Unstable pagination — no sort tiebreaker | 🟡 Medium | Phase 3 | EHS-33 | ✅ Fixed |
| 4 | No pageSize cap — memory bomb | 🟡 Medium | Phase 3 | EHS-33 | ✅ Fixed |
| 5 | No optimistic concurrency — silent data loss | 🔴 High | Phase 4 | EHS-34 | ⬜ Open |
| 6 | AssignedToId/ReportedById — no FK or existence check | 🟡 Medium | Phase 4 | EHS-35 | ⬜ Open |
| 7 | PATCH /assign returns 204 — status change invisible | 🟢 Low | Phase 4/12 | EHS-36 | ⬜ Open |
| 8 | Re-open from Closed unrestricted | 🟢 Low | Phase 4 | EHS-35 | ⬜ Open |
