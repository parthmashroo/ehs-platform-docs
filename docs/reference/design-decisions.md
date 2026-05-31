# Design Decisions & Q&A Reference

> Read when a design question comes up. Not needed at session start.

## CorrelationId Scope — Why Not Link Across Requests?

CorrelationId is per-request scope only. The full user journey is solved by the Phase 6 AuditLog. The audit table records every write by `UserId` in order — each row carries its own CorrelationId as a link to that request's logs.

| Concern | Tool |
|---|---|
| Single request trace | CorrelationId (EHS-47) |
| User journey over time | AuditLog (Phase 6) |
| Cross-service trace | OpenTelemetry (future) |

## Does Logging/Auditing Block the Request?

- **Serilog** — zero blocking cost. Buffers internally, writes async.
- **Phase 6 AuditLog** — small intentional cost. Writes in the same DB transaction as the business write. If the write rolls back, the audit row rolls back too.
- **True async separation** — Phase 9 (Outbox Pattern). Notifications, emails, downstream events write to an Outbox table; a background worker processes them.

## Should Auditing Be a Toggleable Module Per Customer?

Yes, but the mechanism is feature flags / tenant configuration, not separate services. MediatR pipeline behaviours (`AuditBehaviour`, `LoggingBehaviour`) are modular — conditionally skipped per tenant config. A full entitlement system is Phase 14+.

## GDPR & Compliance — Phase 14+ Concern

Key conflicts to resolve:
- **Soft deletes vs Right to Erasure** — pseudonymisation (replace personal identifiers with anonymous tokens, keep audit records intact)
- **Data minimisation** — only collect what's needed
- **Data portability** — export a user's data on request
- **Breach notification** — 72-hour window requirement
- **Admin data access** — reading other users' data (even read-only) is a GDPR violation in many jurisdictions. No impersonation or shadow mode without legal review.

## Localisation / i18n — Phase 12+ Concern

All hardcoded error messages need to go through a resource file when localisation is added. Flag every hardcoded user-facing string for this phase.

## Resource-Level Authorization — Phase 5/6 Concern

Users who are named on a record (author, approver, assignee) must retain permanent read access regardless of role changes:
```
canView = hasRolePermission OR isAuthor OR isApprover OR isAssignee
```

## Role Change & Audit Trail

When a user's role changes, historical records they created/approved remain attributed to them permanently. Role changes do not rewrite history. Implement via immutable `ChangedById` on AuditLog — the name is resolved from User at read time.

## Permissions Layer (Phase 8+)

One role per user is correct for Phase 4. When needed:
- Permissions layer sits on top of roles
- A role is a preset bundle of permissions
- Admins can add/remove individual permissions per user (overrides)
- Do NOT add multiple roles per user — use permission overrides instead

---

## EHS-52 Spike — Elasticsearch vs SQL Full-Text Search

**Date:** May 2026 | **Confidence:** High

| | Elasticsearch | SQL Full-Text Search |
|---|---|---|
| Setup to first query | ~5 minutes | Full-Text Catalog + Index + background population job required first |
| Relevance scoring | Yes — ranked automatically | No — all matching rows equal |
| Multi-field search | Native | More verbose syntax |
| Zero-setup for new fields | Yes | Must recreate Full-Text Index |

**SQL Full-Text result:** `Msg 7601: Cannot use CONTAINS predicate because Incidents is not full-text indexed.`

**Verdict:** Elasticsearch is the correct long-term search layer. SQL Full-Text stays through Phase 10. Switch evaluation at Phase 11.

---

## DateTimeOffset — Open Decisions (EHS-62 Architecture Review)

### JSON Output: Z Suffix vs +00:00
AWS, Azure, and GCP all emit `Z`. System.Text.Json defaults to `+00:00`. A `DateTimeOffsetUtcZConverter` is written and ready. Decision: **emit Z**. Add converter to `Program.cs` before EHS-58 ships.

### Named Timezone Storage
Storing only `DateTimeOffset` (offset) is insufficient at DST boundaries. Australia flips DST — an incident at 2:30 AM on clock-change night has an ambiguous offset (+10 or +11). Without the IANA zone name `"Australia/Sydney"`, the correct local time is unrecoverable.

**Decision:** Add `string? IncidentTimeZoneId nvarchar(50)` to `Incident` entity alongside `OccurredAt`. Store IANA zone name (e.g., `"Australia/Sydney"`). Use `TimeZoneInfo.FindSystemTimeZoneById(tzId)` for display — works cross-platform in .NET 8 without any library. No NodaTime needed.

### UTC Normalization at API Boundary
Validators currently check `LessThanOrEqualTo(DateTimeOffset.UtcNow)` but do not check `Offset == TimeSpan.Zero`. A client can submit `OccurredAt: 2026-05-31T09:00:00+05:30` and it persists with a `+05:30` offset. Add `.Must(x => x.Offset == TimeSpan.Zero)` to `OccurredAt` and `DueDate` validators.
