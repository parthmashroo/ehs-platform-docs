# Phase 6 — Audit Logging: Status & Notes

> Read this when working on any Phase 6 ticket.

## Tickets

| Ticket | Description | Status |
|---|---|---|
| EHS-56 | AuditLog entity + EF config + migration | ✅ Done |
| EHS-57 | AuditInterceptor (SaveChangesInterceptor) — before/after JSON on all auditable entities | ✅ Done |
| EHS-62 | Global DateTime → DateTimeOffset sweep — BaseEntity, all entities, commands, validators, DTOs, AuditInterceptor, migration | ✅ Done |
| EHS-58 | GET /api/incidents/{id}/audit-log + GET /api/correctiveactions/{id}/audit-log | ⬜ Next |
| EHS-59 | Phase 6 docs update | ⬜ To Do |

## EHS-58 Design Notes (read before starting)

- AuditLog has **no Global Query Filter** (no `ITenantEntity`) — queries MUST manually filter by `TenantId`. Do not forget this.
- Response should return **field-level diffs**, not raw `OldValues`/`NewValues` JSON blobs — tracked as technical-debt.md #23.
- `ChangedById` is a FK to `Users` — resolve display name at read time, do not store inline.
- Scope: GET by entity id, scoped to tenant, paginated, ordered by `ChangedAt` descending.

## Phase 6 Lessons Learned

- `SaveChangesInterceptor` hooks into `SavingChanges` — NOT `SavedChanges`. For Guid PKs (client-assigned), the entity ID is available before the DB write, so audit rows can be built and added to ChangeTracker in `SavingChanges`. Both rows commit in the same transaction.
- Enum serialization: use `p.Metadata.ClrType` (schema-first) to detect enums, then `Nullable.GetUnderlyingType()` to unwrap nullable enums. Do not use `p.CurrentValue is Enum e` — runtime pattern match on null is coincidentally correct, not by design.
- `IsConcurrencyToken` check skips RowVersion from audit JSON — byte[] RowVersion columns must be excluded.
- DI wiring: `AddDbContext<T>((sp, options) => ...)` factory overload is required to resolve a scoped interceptor from DI. The simple `AddDbContext(options => ...)` has no `IServiceProvider` access.
- Pin `Microsoft.EntityFrameworkCore.InMemory` to `8.0.0` when adding via CLI — without the version flag, `dotnet add package` resolves latest (10.x), which targets net10 and fails on net8.
- Architecture gap: `TenantId == Guid.Empty` on authenticated user → audit rows written with empty TenantId silently. Tracked in technical-debt.md.
- Test coverage gap: `CorrectiveAction` auditing not yet tested — `AuditableTypes` includes it but no test verifies `EntityName = "CorrectiveAction"`.

## Open Decisions (from EHS-62 architecture review)

| Decision | Status |
|---|---|
| JSON output format — emit Z suffix, not +00:00 | Converter written, not yet committed |
| UTC normalization at validators — `.Must(x => x.Offset == TimeSpan.Zero)` | Not yet implemented |
| Named timezone storage — add `IncidentTimeZoneId nvarchar(50)` to Incident | Ticket not yet created |
| Testcontainers round-trip test for datetimeoffset column | Ticket not yet created |
