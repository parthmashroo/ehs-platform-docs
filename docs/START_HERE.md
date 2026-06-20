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

**92 tests green (as of EHS-72). No new code committed since.**

---

## Phase 7 — Remaining Sprint (verified from Jira 2026-06-20)

> This section replaces ad-hoc "next ticket" notes. Read this every session. Update status as tickets complete.

### ⚠️ Duplicate to close
EHS-80 (created Session 32) = EHS-77 (already existed from multi-reviewer audit). **Close EHS-80 as duplicate.** Work on EHS-77.

### Session A — Quick security wins (~2.5 hrs total, 3 commits)

| Ticket | What | Effort | Status |
|---|---|---|---|
| **EHS-77** | Suppress `exception.Message` on 500 — single line in `ExceptionHandlingMiddleware` + 1 test | 30 min | ⬜ TODO |
| **EHS-76** | AuditLog global query filter — add `ITenantEntity` to AuditLog, add filter in DbContext, remove 2 manual WHERE clauses in handlers | 1 hr | ⬜ TODO |
| **EHS-74** | JWT signing key out of source — user-secrets (dev), env var (prod), startup guard, pin `ValidAlgorithms` | 1 hr | ⬜ TODO |

### Session B — Migrations (~3 hrs total, 2 commits)

| Ticket | What | Effort | Status |
|---|---|---|---|
| **EHS-75** | `HasColumnType("datetimeoffset")` on every `DateTimeOffset` property in all entity configs + migration | 2 hrs | ⬜ TODO |
| **EHS-78** | Fix audit index mismatch + composite tenant indexes (`TenantId, Status, OccurredAt DESC WHERE IsDeleted=0`) — migration only | 1 hr | ⬜ TODO |

### Session C — Tenant isolation seam (~4 hrs, own session, highest risk)

| Ticket | What | Effort | Status |
|---|---|---|---|
| **EHS-73** | User global query filter + TenantId, TenantStampInterceptor, stamp TenantId on CreateIncident + CreateCorrectiveAction, AuditLog filter (overlaps EHS-76 — coordinate) | 4 hrs | ⬜ TODO |

### Session D — Quality + architecture (after P0/P1 cleared)

| Ticket | What | Effort | Status |
|---|---|---|---|
| **EHS-79** | Code-quality bundle: ClaimNames constants, ctype wiring, shared audit-query builder, backfill tests | 2-3 hrs | ⬜ TODO |
| **EHS-81** | ACL wrapper interfaces — Proprietary Vocabulary for all non-.NET frameworks | 1 session | ⬜ TODO |
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
