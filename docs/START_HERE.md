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

**Session 26 (2026-06-06):** EHS-65 committed. `IAuditableEntity` marker interface added to Domain.Common. `Incident` and `CorrectiveAction` implement it. `AuditInterceptor` hardcoded `HashSet<Type>` removed — replaced with `is not IAuditableEntity` check. 62/62 tests green. Interview card Q62 added.

**Phase 6 complete — all tickets done:**
- EHS-56: AuditLog entity + EF config + migration ✅
- EHS-57: AuditInterceptor — SavingChanges snapshot + ChangeTracker add pattern ✅
- EHS-62: DateTime → DateTimeOffset global sweep + UTC validator + `DateTimeOffsetUtcZConverter` ✅
- EHS-58: GET audit-log endpoints, policy-driven auth, config-driven roles, soft-deleted user names ✅
- EHS-59: Phase 6 docs update ✅

**Phase 7 in progress:**
- EHS-69: CORS — named policy, config-driven origins, API integration test project ✅
- EHS-65: `IAuditableEntity` marker interface — opt-in auditing, replaced hardcoded type registry ✅

**Open technical debt (in technical-debt.md):**
- #25: `a.Action.ToString()` inside LINQ select — client-side evaluation risk
- #26: `IgnoreQueryFilters()` undocumented scope — future TenantId filter bypass risk
- #27: `TenantId == Guid.Empty` guard missing in both audit log handlers
- #28: CORS integration test relies on Development env config ✅ Fixed
- #29: CORS `WithHeaders` allow-list has no maintenance process

**62 tests green. Next: pick next Phase 7 ticket from Jira.**

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
- Transition Jira → In Progress when starting, → Done when committed
- Tests ship with every commit — never deferred
- After every commit: run the 7-question architecture review (see CLAUDE.md)
