# Phase 7 — Hardening & Multi-Reviewer Audit: Status & Notes

> Read this when working on any Phase 7 ticket. Phase 6 (Audit Logging) is complete — see `phase-6-status.md` archived context.

## Tickets

| Ticket | Description | Status |
|---|---|---|
| EHS-69 | CORS middleware — named policy, config-driven origins, API integration test project | ✅ Done |
| P0-1 | MediatR ValidationBehavior — validators currently dead code | ⬜ Backlog |
| P0-2 | Tenant isolation seam — User filter + TenantStampInterceptor + CA parent check | ⬜ Backlog |
| P0-3 | Move JWT signing key out of source + rotate + pin algorithm | ⬜ Backlog |
| P0-4 | datetimeoffset column-type migration (all entities) | ⬜ Backlog |
| P1-x | AuditLog global filter · 500 message leak · concurrency (EHS-55) · indexes · write/delete authz | ⬜ Backlog |
| P2 | Quality bundle — claim constants, ctype wiring, audit-handler dedup, test backfill | ⬜ Backlog |

> Ticket numbers above are placeholders — assign real EHS-xx in Jira. Full descriptions + acceptance criteria captured in the review session; debt detail in `technical-debt.md` (Phase 7 Multi-Reviewer Audit section, rows 30-38).

## Multi-Reviewer Audit (Jun 2026)

Whole-codebase review by 4 independent agents — architecture, security, database, code-quality. Key value: **independent convergence**. Where 3 separate reviewers, given no shared context, land on the same finding, confidence is near-certain.

### What converged (highest confidence)
- **Tenant isolation is not structurally enforced** (Arch + Security + DB, all 3): `User`/`Organization` query filter omits `TenantId`; `TenantId` not stamped on Incident/CorrectiveAction create; no interceptor; `User` doesn't implement `ITenantEntity`. → P0-2.
- **All FluentValidation validators are dead code** (Arch + Code): no `IPipelineBehavior` runs them. → P0-1.
- **RowVersion concurrency is inert** (Arch + DB): confirms existing EHS-55.
- **Duplicated validation rules** across all 4 Create/Update validators (Arch + Code): confirms existing shared-rules ticket — concrete list now appended to that ticket.
- **AuditLog lacks a global query filter** (Arch + DB): isolation hand-rolled per query.

### Single-reviewer, high-impact
- **JWT signing key committed in appsettings.json** (Security) — cross-tenant token forgery. Rotate the key. → P0-3.
- **SQL columns are `datetime2`, not `datetimeoffset`** (DB) — distinct from EHS-62 (which fixed the C# type). Offset discarded at persistence layer. → P0-4.
- **`exception.Message` leaked on 500** (Security) — schema/SQL disclosure.

### What came back clean
- Clean Architecture project-reference direction intact — Domain has zero dependencies, no reverse references.
- CQRS 3-file folder convention followed consistently.
- CORS (EHS-69) uses an explicit origin allowlist, not a wildcard; pairs `AllowCredentials` safely.
- Audit-log endpoints (EHS-58) correctly tenant-scope their reads.

## Lessons / Process

- Multi-agent independent review is high-signal when agents converge — treat 3-way agreement as a near-certainty, single-reviewer findings as leads to verify.
- The recurring theme across P0s: **cross-cutting concerns (tenant stamping, soft-delete, validation, concurrency) are enforced by per-handler discipline, not by interceptors/behaviors.** The structural fix (one interceptor/behavior each) removes a whole class of "the next handler forgot" bugs. This is the dominant Phase 7 architectural direction.

---

## Open Decisions

| Decision | Status |
|---|---|
| Sequence P0s — recommend P0-2 (tenant seam) first, it's the security spine | ⬜ Pending |
| `ctype` claim — wire into an authz decision now, or formally defer with a ticket | ⬜ Pending |
| RowVersion — finish HTTP contract (EHS-55) or remove token + 409 to avoid false confidence | ⬜ Pending |
| Carried from Phase 6: `IncidentTimeZoneId` column + migration; Testcontainers datetimeoffset round-trip test | ⬜ Tickets needed |
