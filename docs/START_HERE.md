# EHS Platform — AI Bootstrap

Read in this order:

1. `MEMORY.md` — user profile and project context (persisted memory)
2. `dev-notes.md` — current phase status, all tickets, lessons learned (source of truth)
3. `project-roadmap.md` — all 17 phases, architecture decisions, full scope

---

## Current Status (as of May 2026)

Phase 0 ✅ Phase 1 ✅ Phase 2 ✅ Phase 3 ✅ Phase 4 ✅ Phase 5 🔄 IN PROGRESS

Active sprint: Sprint 4.
Next ticket: **EHS-50** — EF Core Global Query Filters + TenantResolutionMiddleware.

---

## Key Documents

| Document | Purpose |
|---|---|
| `dev-notes.md` | Ticket-by-ticket status, code patterns, lessons learned |
| `project-roadmap.md` | All 17 phases, full scope, technology stack |
| `architecture-decisions.md` | All ADRs — locked decisions, never revisit |
| `technical-debt.md` | Known debt, deferred decisions |
| `ai-first-strategy.md` | Phase 17 AI architecture vision — MCP, AI Service Principal |
| `ai-capabilities-research.md` | Market research — what competitors do, what practitioners want |
| `ai-strategy-session-handoff.md` | All 7 AI capability gaps — verdicts and decisions locked |
| `semantic-form-engine-design.md` | Phase 7 form engine — semantic AI layer, scoring, pre-built templates |
| `maisaas-deep-analysis.md` | Reference system analysis (prior system Parth worked on) |
| `career-profile.md` | Developer background and goals |

---

## Rules (Non-Negotiable)

- Never edit files in `c:\Projects\ehs-platform` directly — tell Parth what to change
- Doc changes in `C:\Projects\ehs-platform-docs` — make directly, commit, push
- All architecture decisions are locked — never re-debate them
- Check Jira (pmashroo.atlassian.net) before starting any ticket
- Tests ship with every commit — never deferred
- No Claude attribution in commits, PRs, or Jira
