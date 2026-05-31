# EHS Platform — Session Start

**Read this file every session. Read nothing else unless this file tells you to.**

---

## Current State

Phase 6 🔄 IN PROGRESS | Sprint 7 | Next ticket: **EHS-58**

EHS-58: GET /api/incidents/{id}/audit-log + GET /api/correctiveactions/{id}/audit-log

---

## Last Session Handoff

**Session 22 (2026-05-31):** EHS-62 carry-overs committed. Z-suffix converter wired, UTC offset validation added to all 4 validators. 52 tests green.

**Completed this session:**
- `DateTimeOffsetUtcZConverter` wired in `Program.cs` — all DateTimeOffset fields now serialize as `Z` suffix
- UTC normalization enforced at all 4 validators (OccurredAt × 2, DueDate × 2) — `.Must(x => x.Offset == TimeSpan.Zero)`
- Test files cleaned up: `DateTime.UtcNow` → `DateTimeOffset.UtcNow` across 3 test files
- technical-debt.md #24 marked ✅ Fixed

**Still needed (not yet in Jira):**
- Named timezone on Incident: `string? IncidentTimeZoneId nvarchar(50)` + migration
- Testcontainers round-trip test for datetimeoffset column type

**Watch out for EHS-58:**
- AuditLog has NO Global Query Filter — queries MUST manually filter `WHERE TenantId = @tenantId`
- Response should return field-level diffs, not raw JSON blobs (technical-debt.md #23)
- `ChangedById` → resolve display name at read time, do not store inline

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
