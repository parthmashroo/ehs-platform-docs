# EHS Platform — Session Start

**Read this file every session. Read nothing else unless this file tells you to.**

---

## Current State

Phase 6 ✅ COMPLETE | Phase 7 🔄 NEXT | Sprint 8

Phase 7: Semantic Form Engine. Create Phase 7 Jira tickets before coding starts.

**Backlog items needing Jira tickets:**
- `IncidentTimeZoneId nvarchar(50)` on Incident + migration (named timezone storage)
- Testcontainers round-trip test for `datetimeoffset` column type

---

## Last Session Handoff

**Session 38 (2026-06-27):** EHS-73 committed. Closed the tenant isolation seam: `User` implements `ITenantEntity` + `HasQueryFilter(TenantId == _tenantId && !IsDeleted)`. `LoginCommandHandler` and `RegisterCommandHandler` use `IgnoreQueryFilters()` + manual `!u.IsDeleted` for pre-auth paths. New `TenantStampInterceptor` (SaveChangesInterceptor) auto-stamps `TenantId` on all `Added` `ITenantEntity` rows. `CreateIncidentCommandHandler` explicitly sets `TenantId`. `CreateCorrectiveActionCommandHandler` injects `ICurrentUserService`, adds IDOR check (`Incidents.AnyAsync` owns correct tenant) and explicit `TenantId` stamp. `ICurrentUserService` made required in `ApplicationDbContext` constructor (removed `= null` default). 8 test files updated for required constructor; 1 new `TenantStampInterceptorTests` (5 tests); `TenantIsolationTests` rewritten (3 cross-tenant rejection tests). 106/106 tests green. Interview card Q73 added. Debt #31-#33, #35-#36 ✅. Debt #51-#55 added.

**Session 37 (2026-06-21):** EHS-78 committed. Added composite filtered indexes: `(TenantId, Status, OccurredAt DESC WHERE IsDeleted=0)` on Incidents, `(TenantId, Status, DueDate WHERE IsDeleted=0)` on CorrectiveActions. Fixed AuditLog composite indexes to use `IsDescending(..., true)` on ChangedAt — prior indexes had ChangedAt ASC, all queries used OrderByDescending (full in-memory sort on every query). Bare single-column `IX_Incidents_TenantId` and `IX_CorrectiveActions_TenantId` dropped by EF — composite leading column covers FK lookup (GitHub #33454). 4 new model metadata tests; 2 use `IDesignTimeModel` for IsDescending (runtime model strips design-time metadata). 98/98 tests green. Interview card Q72 added. Debt #49, #50 added.

**Session 36 (2026-06-21):** EHS-75 committed. Overrode `ConfigureConventions` in `ApplicationDbContext` to call `HaveColumnType("datetimeoffset")` for `DateTimeOffset` and `DateTimeOffset?` globally — prevents EF Core default `datetimeoffset(7)` string from diverging against snapshot's `datetimeoffset`, which caused spurious ALTER COLUMN on every migration add. Added missing explicit configs for `CompletedAt` (CorrectiveAction) and `LastLoginAt` (User). Added model metadata test iterating all EF entity types and asserting column type = `"datetimeoffset"`. DB verified via SQL MCP — 21/21 DateTimeOffset columns already `datetimeoffset(7)`. Migration was empty (schema already correct from EHS-62). 94/94 tests green. Interview card Q71 added.

**Session 35 (2026-06-21):** EHS-74 committed. Removed `SecretKey` from `appsettings.json`. Added `UserSecretsId` to API .csproj via `dotnet user-secrets init`. Set rotated dev key via user-secrets (old key was compromised — in Git history). Added startup guard in `DependencyInjection.AddInfrastructure()` — throws `InvalidOperationException` on missing or sub-32-char key with actionable fix message. Fixed `WebApplicationFactory` integration tests (`CorsIntegrationTests`, `CreateIncidentValidationIntegrationTests`) to inject test key via `AddInMemoryCollection`. 93/93 tests green. Interview card Q70 added.

**Session 33 (2026-06-20):** EHS-76 committed. Added `ITenantEntity` to `AuditLog`, `HasQueryFilter(x => x.TenantId == _tenantId)` in `ApplicationDbContext`. Removed manual `where a.TenantId == tenantId` from both audit log query handlers — split into two queries (AuditLog fetch + Users dict lookup) to isolate `IgnoreQueryFilters()` scope (in-memory provider leaks hint to entire LINQ tree; SQL Server does not). Tests updated to pass mock `ICurrentUserService` into `ApplicationDbContext` constructor. 93/93 green. Debt #34 ✅, #48 added. Interview card Q69 added.

**Session 32 (2026-06-20):** EHS-77 committed. Suppressed `exception.Message` in `ExceptionHandlingMiddleware` for 500 responses — `Detail = statusCode == HttpStatusCode.InternalServerError ? null : exception.Message`. Domain exceptions (404/422/403/409) still surface their authored messages. 1 new unit test using `DefaultHttpContext` + `NullLogger` asserts 500 body never contains internal detail. 93/93 tests green. Debt: `DbUpdateConcurrencyException` (409) also leaks EF entity names — tracked. Interview card Q68 added.

**Session 31 (2026-06-15):** EHS-72 committed. Wired FluentValidation to MediatR pipeline via `ValidationBehavior<TRequest,TResponse>` in `Application/Common/Behaviours/` — all `*CommandValidator` classes were dead code before this. ExceptionHandlingMiddleware upgraded to return `ValidationProblemDetails` with per-field errors. 24 new tests (behavior unit, validator unit ×2, integration test asserting 400 + field errors). 92/92 tests green. Debt #30 ✅, #45 + #46 added. Interview card Q67 added.

**Session 30 (2026-06-13):** EHS-71 committed. Extracted `MustBeUtc()` FluentValidation extension to `Application/Common/Validation/CommonValidatorRules.cs` — eliminates inline `x.Offset == TimeSpan.Zero` predicate duplicated across 4 validators. 68/68 tests green (4 new isolation tests for shared rule). Debt #43, #44 added. Interview card Q66 added.

**Session 29 (2026-06-13):** EHS-70 committed. Fixed `a.Action.ToString()` inside LINQ expression tree in both audit log query handlers — was potential EF Core client-side eval. Two-step fix: `(int)a.Action` in SQL, `((AuditAction)r.ActionInt).ToString()` in-memory after ToListAsync. 64/64 tests green. Debt #25 ✅. Interview card Q65 added.

**Session 28 (2026-06-07):** EHS-67 committed. Added CorrectiveAction audit test — pins `EntityName = "CorrectiveAction"`, `Action = Created`, `TenantId`, `ChangedById`. 64/64 tests green. Arch finding: `GetType().Name` proxy-unsafe → fixed same session (`entry.Metadata.ClrType.Name`, debt #42 ✅). Interview card Q64 added.

**Session 27 (2026-06-06):** EHS-66 committed. `AuditInterceptor.AddAuditLogs()` guard extended — returns early when `TenantId == Guid.Empty` even if `IsAuthenticated = true`. Prevents audit rows with empty TenantId from poisoning tenant-scoped queries. One new test added (authenticated + empty tenant → no audit log). 63/63 tests green. Interview card Q63 added.

**Phase 6 complete — all tickets done:**
- EHS-56: AuditLog entity + EF config + migration ✅
- EHS-57: AuditInterceptor — SavingChanges snapshot + ChangeTracker add pattern ✅
- EHS-62: DateTime → DateTimeOffset global sweep + UTC validator + `DateTimeOffsetUtcZConverter` ✅
- EHS-58: GET audit-log endpoints, policy-driven auth, config-driven roles, soft-deleted user names ✅
- EHS-59: Phase 6 docs update ✅

**Phase 7 in progress:**
- EHS-69: CORS — named policy, config-driven origins, API integration test project ✅
- EHS-65: `IAuditableEntity` marker interface — opt-in auditing, replaced hardcoded type registry ✅
- EHS-66: AuditInterceptor TenantId == Guid.Empty guard ✅
- EHS-67: CorrectiveAction audit interceptor coverage ✅
- EHS-70: Audit log handlers — fix `Action.ToString()` client-eval ✅
- EHS-71: Extract `MustBeUtc()` shared validator rule — 4 validators updated ✅
- EHS-72: Wire ValidationBehavior to MediatR pipeline — dead validators are now live ✅

**106 tests green (as of EHS-73).**

---

## Phase 7 — Remaining Sprint (verified from Jira 2026-06-20)

> This section replaces ad-hoc "next ticket" notes. Read this every session. Update status as tickets complete.

### ⚠️ Duplicate to close
EHS-80 (created Session 32) = EHS-77 (already existed from multi-reviewer audit). **Close EHS-80 as duplicate.** Work on EHS-77.

### Session A — Quick security wins (~2.5 hrs total, 3 commits)

| Ticket | What | Effort | Status |
|---|---|---|---|
| **EHS-77** | Suppress `exception.Message` on 500 — single line in `ExceptionHandlingMiddleware` + 1 test | 30 min | ✅ DONE |
| **EHS-76** | AuditLog global query filter — add `ITenantEntity` to AuditLog, add filter in DbContext, remove 2 manual WHERE clauses in handlers | 1 hr | ✅ DONE |
| **EHS-74** | JWT signing key out of source — user-secrets (dev), env var (prod), startup guard | 1 hr | ✅ DONE |

### Session B — Migrations (~3 hrs total, 2 commits)

| Ticket | What | Effort | Status |
|---|---|---|---|
| **EHS-75** | `HasColumnType("datetimeoffset")` on every `DateTimeOffset` property in all entity configs + migration | 2 hrs | ✅ DONE |
| **EHS-78** | Fix audit index mismatch + composite tenant indexes (`TenantId, Status, OccurredAt DESC WHERE IsDeleted=0`) — migration only | 1 hr | ✅ DONE |

### Session C — Tenant isolation seam (~4 hrs, own session, highest risk)

| Ticket | What | Effort | Status |
|---|---|---|---|
| **EHS-73** | User global query filter + TenantId, TenantStampInterceptor, stamp TenantId on CreateIncident + CreateCorrectiveAction, AuditLog filter (overlaps EHS-76 — coordinate) | 4 hrs | ✅ DONE |

### Session D — Quality + architecture (after P0/P1 cleared)

| Ticket | What | Effort | Status |
|---|---|---|---|
| **EHS-79** | Code-quality bundle: ClaimNames constants, ctype wiring, shared audit-query builder, backfill tests | 2-3 hrs | ⬜ TODO |
| **EHS-81** | ACL wrapper interfaces — Proprietary Vocabulary for all non-.NET frameworks | 1 session | ⬜ TODO — **PROMOTED: P1, do after EHS-74, not last. 45 files already import MediatR/FV directly. 8 files added in 1 sprint. Deferral cost compounds.** |
| **EHS-82** | Write ADR-017 doc (MediatR license risk) | 30 min | ⬜ TODO |
| **EHS-83** | Write ADR-018 doc (ACL-First principle) | 45 min | ⬜ TODO |

### Backlog — defer, not this phase

| Ticket | Defer to |
|---|---|
| EHS-61: NetArchTest architecture layer tests | Phase 8 |
| EHS-55: ETag/If-Match HTTP contract | Phase 5 leftover — Phase 8 |
| EHS-54: Revoke JWT on soft-delete | Phase 8 |
| EHS-60: GDPR erasure strategy | Phase 9+ |
| EHS-63: Replace DbSet→IQueryable in IApplicationDbContext | Phase 8 |
| EHS-64: DB-managed timestamps + RowVersion shadow | Phase 8 |
| EHS-68: Permission-Based RBAC | Phase 8 |

**Start every session at the top of the first incomplete Session block above. Do not jump ahead.**

---

## What to Read Next

Only read a second file if you actually need it:

| Task | Read |
|---|---|
| Working on Phase 6 tickets | `docs/current/phase-6-status.md` |
| Making an architectural decision | `docs/architecture-decisions.md` (search for the relevant ADR) |
| Adding a new entity | `docs/reference/patterns.md` |
| Auth / JWT questions | `docs/reference/auth-jwt.md` |
| Design Q&A, future phase notes, ES spike | `docs/reference/design-decisions.md` |
| Known debt | `docs/technical-debt.md` |
| All 17 phases scope | `docs/project-roadmap.md` |
| Phase 1–5 ticket history | `docs/archive/dev-notes-phases-1-5.md` |
| Full session log | `docs/archive/session-log.md` |
| Project vitals, NuGet, DB, middleware | `docs/dev-notes.md` |

---

## Non-Negotiable Rules (full rules in CLAUDE.md)

- Never edit files in `c:\Projects\ehs-platform` — tell Parth exactly what to change
- Doc changes in `C:\Projects\ehs-platform-docs` — Claude makes these directly, commits, pushes
- Read this C:\Users\Komal\.claude\projects\c--Projects-ehs-platform\memory\feedback_docs_ownership.md
- Transition Jira → In Progress when starting, → Done when committed
- Tests ship with every commit — never deferred
- After every commit: run the 7-question architecture review (see CLAUDE.md)
