# EHS Platform — AI-First Strategy

> Written May 2026. 18-month forward view. Not a features list — an architectural position.
>
> This is not "AI as a feature." This is the platform built from the inside as an AI-native system.
>
> **Companion documents (read together):**
> - `ai-capabilities-research.md` — market research, competitor analysis, practitioner survey data
> - `ai-strategy-session-handoff.md` — all 7 AI capability gaps, verdicts locked
> - `semantic-form-engine-design.md` — Phase 7 form engine AI-readability design
> - `architecture-decisions.md` ADR-014 — locked AI architecture decisions

---

## The Distinction That Matters

Most software companies say "AI-powered" and mean:
- A chatbot in the corner of the UI
- A button that summarises a report
- A classifier bolted onto an existing form

That is **AI as decoration.**

We are building something different: a platform whose data model, API surface, event pipeline,
and integration layer are designed to be **operated by AI agents**, not just consumed by humans
through a UI.

The UI is one client. AI agents are another. Both are first-class.

---

## What Changes in 18 Months (May 2026 → late 2027)

AI capabilities are not growing linearly. By late 2027:

| Capability | Today (May 2026) | 18 Months Out |
|---|---|---|
| Context window | 200k tokens | 1M+ tokens standard |
| Tool use / MCP | Early adoption | Enterprise standard |
| Reasoning depth | Strong | Near-expert in narrow domains |
| Agentic loops | Experimental | Production-grade |
| Multi-agent coordination | Research | Shipping in enterprise tooling |
| Cost per token | Dropping ~50% per year | 4× cheaper than today |

The enterprise buyers we are targeting (construction, manufacturing, mining, energy) will
have AI assistants embedded in their workflows by then. Every major vendor — SAP, Oracle,
Microsoft, Salesforce — will have shipped AI copilots. The question is not whether AI
will be in the workplace. The question is **whether our platform is native to that world
or retrofitted into it.**

Retrofitting costs twice as much and ships a year late. We are building native.

---

## The MCP Layer — EHS Platform as an AI-Callable Server

**Model Context Protocol (MCP)** is Anthropic's open standard for giving AI agents structured
access to external systems. An MCP server exposes typed tools that any compatible AI assistant
can call — Claude, Microsoft Copilot, any future model.

We build an EHS MCP Server as a thin layer over our existing REST API.

### What this looks like technically

```
MCP Tools exposed by EHS Platform:
─────────────────────────────────────────────────────────
create_incident(tenantId, siteId, type, severity, description)
get_incidents(tenantId, filters: status, severity, dateRange)
get_incident_by_id(tenantId, incidentId)
assign_incident(tenantId, incidentId, userId)
transition_incident_status(tenantId, incidentId, newStatus)

create_corrective_action(tenantId, incidentId, title, assignedTo, dueDate)
get_overdue_corrective_actions(tenantId, siteId?)
transition_corrective_action(tenantId, caId, newStatus)

get_site_risk_summary(tenantId, siteId, dateRange)
get_open_permits(tenantId, siteId?)
get_compliance_status(tenantId)
─────────────────────────────────────────────────────────
```

Every tool is authenticated, tenant-scoped, and runs through the exact same MediatR
pipeline — validation, audit logging, CorrelationId — as a human HTTP request. The MCP
layer is just another client. No special path, no bypassed rules.

### What this enables

A safety manager using their enterprise AI (whatever model the company has adopted):

> *"What are the top 3 open corrective actions at Site B that are past due, and draft
> an email to the site lead summarising them?"*

The AI calls `get_overdue_corrective_actions`, gets structured data, synthesises it,
drafts the email. No dashboard. No BI tool. No report builder. No login to a UI.

An AI agent running on a schedule:

> Every Monday at 07:00 — call `get_compliance_status` for all tenants, identify any
> with open critical incidents older than 14 days, send flagged summary to the compliance officer.

This is not a new feature. It is the same data, the same rules, the same API — just
callable by an agent instead of a human.

### What it costs us to build

The MCP server is a **thin adapter** over our existing API. Because we built Clean
Architecture from day one:
- The MediatR pipeline is already the execution path
- The MCP tool handlers call the same Commands and Queries as the HTTP controllers
- Tenant isolation, validation, and audit logging come for free
- No new business logic. Just a new transport.

Estimated effort when we reach Phase 17: 2–3 weeks for a production-quality MCP server
covering the core EHS domain.

---

## ERP / CRM Integration — AI as the Integration Layer

Traditional ERP integration is point-to-point: our API ↔ SAP API. Field mappings hardcoded.
Breaks when either side changes. Costs months to build per ERP.

The AI-native alternative:

```
SAP MCP Server ←→ AI Orchestrator ←→ EHS MCP Server
```

The AI becomes the integration layer. Instead of hardcoded field mappings, you write
intent: *"When an EHS incident involves an asset, create a maintenance notification in SAP PM."*

The AI knows what an "incident" is in our domain. It knows what a "maintenance notification"
is in SAP's domain. It maps the intent, not the fields.

### Why this matters for ERP specifically

The ERP integration point that creates real enterprise lock-in is **Work Permit Management**
(Phase 13). A permit-to-work is the gate on every maintenance activity. When our permit
system talks to the ERP's maintenance scheduling (SAP PM, IFS, Oracle EAM):

- A maintenance job cannot start without a valid permit in our system
- When a permit is issued, the ERP work order status updates
- When a permit is closed (job done), the ERP updates maintenance history
- When an incident occurs on a permitted job, both systems record it

This is not a nice-to-have. In safety-critical industries (oil & gas, mining, nuclear),
this linkage is a **regulatory requirement**. No standalone EHS tool does it well today.
We are building the data model (Phase 13) and the transport (Phase 17 MCP) that makes
this integration native.

### The CRM angle

CRM integration is secondary but real:
- Contractor management (Phase 13) → contractor records in Salesforce/Dynamics
- Incident → insurance claim trigger (CRM case creation)
- Client account health: incident rate, compliance score surfaced in CRM

The mechanism is the same MCP layer. The enterprise AI orchestrates it.

---

## Architecture Implications — What We Already Have Right

| What We Built | Why It Matters for AI-First |
|---|---|
| Clean Architecture + CQRS | MCP tools map 1:1 to Commands/Queries. No new logic layer. |
| Typed domain model | AI maps our types precisely. Form-engine-based tools cannot. |
| Outbox Pattern (Phase 9) | AI agents trigger actions reliably. No lost events if agent retries. |
| Multi-tenancy row-level | AI can be scoped to one tenant's data per call. |
| CorrelationId on every request | AI agent actions are traceable in audit logs like any human action. |
| Three audit tables (Phase 6) | AI can query audit history as structured data — not scraping logs. |
| `ICurrentUserService` abstraction | AI agent calls are attributable to a service principal, not "system". |

None of this needs to change. The foundation is right.

---

## What We Add in Phase 17 (Revised Scope)

The original Phase 17 scope (classification, pattern detection, risk scoring, report generation)
remains. We expand it:

### Original (kept)
- Incident classification: AI suggests Type and Severity from description
- Pattern detection: clustering similar incidents across sites
- Risk scoring: likelihood of severity escalation
- Compliance report generation: narrative summary, streaming
- Form template generator from natural language description

### Added — AI-Native Infrastructure
- **EHS MCP Server** — production MCP server over the full API surface
- **AI Service Principal** — a special user type that represents an AI agent in the system.
  All AI-triggered actions are attributed to this principal in the audit log.
  Scope-limited by tenant + permitted tool list.
- **Webhook/Event Stream** — AI agents can subscribe to domain events
  (incident created, CA overdue, permit expiring) and react without polling.
  Built on Phase 9 Outbox infrastructure.
- **Structured AI Response Schemas** — every query handler that will be called by AI
  returns a response envelope designed for machine consumption (not just human readability).
  This means: consistent field names, no human-only abbreviations, explicit status codes in body.

### Phase 17 Tech Stack Addition
- Claude API (Anthropic .NET SDK) — already planned
- MCP server library for .NET (`ModelContextProtocol` NuGet) — Phase 17
- No new infrastructure — runs in the same API process or as a sidecar

---

## The Competitive Position

In the EHS software market as of 2026:

| Competitor type | AI posture |
|---|---|
| Legacy EHS tools (Intelex, Cority, Enablon) | Adding AI chatbots on top of a 15-year-old data model |
| New entrants | Building "AI-powered forms" — still form engines |
| ERPs with EHS modules | AI copilots that know their ERP's domain, not EHS-specific |
| Us | Typed EHS domain model + MCP-native + ERP-connectable from day one |

The moat is not the AI. The AI is available to everyone. The moat is the **domain model**.

A typed, validated, auditable, tenant-isolated `Incident` with a state machine, linked
`CorrectiveActions`, a `WorkPermit`, and a `SiteArea` — that is data an AI can reason about
with precision. A schema-less form blob is not.

We built the domain first. The AI layer fits cleanly on top. That is the correct order.

---

## One-Line Summary

> We are not adding AI to an EHS platform. We are building an EHS platform that AI agents
> can operate natively — because the domain model, the API, the audit trail, and the
> event pipeline were all designed with that in mind from day one.

---

*Captured from a working session conversation — May 2026. Phase 17 is the implementation. This document is the why.*
