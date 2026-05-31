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

**Session 24 (2026-05-31):** EHS-59 committed and pushed. Phase 6 complete. Interview cards Q51–Q53 added.

**Phase 6 complete — all tickets done:**
- EHS-56: AuditLog entity + EF config + migration ✅
- EHS-57: AuditInterceptor — SavingChanges snapshot + ChangeTracker add pattern ✅
- EHS-62: DateTime → DateTimeOffset global sweep + UTC validator + `DateTimeOffsetUtcZConverter` ✅
- EHS-58: GET audit-log endpoints, policy-driven auth, config-driven roles, soft-deleted user names ✅
- EHS-59: Phase 6 docs update ✅

**Open technical debt from Phase 6 (in technical-debt.md #25–27):**
- #25: `a.Action.ToString()` inside LINQ select — client-side evaluation risk
- #26: `IgnoreQueryFilters()` undocumented scope — future TenantId filter bypass risk
- #27: `TenantId == Guid.Empty` guard missing in both audit log handlers

**60 tests green. Phase 7 starts next.**

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
