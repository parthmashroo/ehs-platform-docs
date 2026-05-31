# EHS Platform — Session Start

**Read this file every session. Read nothing else unless this file tells you to.**

---

## Current State

Phase 6 🔄 IN PROGRESS | Sprint 7 | Next ticket: **EHS-59**

EHS-59: Phase 6 docs update

---

## Last Session Handoff

**Session 23 (2026-05-31):** EHS-58 committed and pushed. 60 tests green. Audit log endpoints live.

**Completed this session:**
- `AuditableEntity` enum in `Domain/Enums/` — replaces magic strings, enables future entitlement gating
- `AuditLogEntryDto` in `Application/Common/Models/`
- `GetIncidentAuditLogQuery` + handler — explicit TenantId filter, LEFT JOIN Users with `IgnoreQueryFilters()`, `"FullName (Inactive)"` for soft-deleted users
- `GetCorrectiveActionAuditLogQuery` + handler — identical pattern
- `Policies` static class in `Application/Common/Constants/` — policy name constant, no magic strings
- Config-driven `AddAuthorization` in `Program.cs` — roles in `appsettings.json`, not in code
- `AuditLogConfiguration`: extended primary index to include `ChangedAt`; added `(TenantId, ChangedById, ChangedAt)` secondary index
- `IncidentsController` + `CorrectiveActionsController`: `GetAuditLog` action behind `AuditLogAccess` policy
- Migration: `EHS-58_AuditLogIndexes`
- 8 new tests: ordering, tenant isolation, active user name, inactive user suffix

**New technical debt added (EHS-58 review):**
- #25: `a.Action.ToString()` inside LINQ select — client-side evaluation risk
- #26: `IgnoreQueryFilters()` undocumented scope — future TenantId filter bypass risk
- #27: `TenantId == Guid.Empty` guard missing in both handlers

**Still needed (not yet in Jira):**
- Named timezone on Incident: `string? IncidentTimeZoneId nvarchar(50)` + migration
- Testcontainers round-trip test for datetimeoffset column type

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
