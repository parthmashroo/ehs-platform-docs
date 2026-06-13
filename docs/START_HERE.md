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

**Open technical debt (in technical-debt.md):**
- ~~#25: `a.Action.ToString()` inside LINQ select~~ ✅ Fixed EHS-70
- #26: `IgnoreQueryFilters()` undocumented scope — future TenantId filter bypass risk
- #27: `TenantId == Guid.Empty` guard missing in both audit log QUERY handlers (interceptor now guarded via EHS-66)
- #29: CORS `WithHeaders` allow-list has no maintenance process

**64 tests green. Next: pick next Phase 7 ticket from Jira (P0-1 ValidationBehavior recommended — highest leverage).**

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
