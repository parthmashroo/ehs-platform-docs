# AI Strategy Discussion — Session Handoff

> Paste this into a new Claude Code session to continue the AI capabilities conversation.
> Last updated: May 2026.

---

## What We Were Doing

We were doing a 1-by-1 honest walkthrough of the 7 AI-related gaps identified in the
research doc (`ai-capabilities-research.md`), examining how our platform tackles each one,
and being transparent about what's genuinely solved vs. what still has gaps.

**We completed Issue 1 of 7.** Issues 2–7 are pending.

---

## Three Docs to Read First (in this order)

All in `C:\Projects\ehs-platform-docs\docs\`:

1. `ai-first-strategy.md` — the architectural vision (AI-native, not AI-decorated)
2. `ai-capabilities-research.md` — full market research: named vendors, NSC survey stats,
   what practitioners want, what not to build, ERP integration landscape
3. `semantic-form-engine-design.md` — the Phase 7 form engine design that solves the
   AI-readability gap for dynamic forms

---

## Issue 1 — Already Discussed (Summary)

**Issue:** Form engine architecture cannot be reasoned about by AI.

**Honest verdict:** Partial advantage for us.
- Core typed entities (Incident, CorrectiveAction, WorkPermit, etc.) — genuine AI advantage.
  State machines are domain-enforced, not just UI validation. Relationships are FK-constrained.
  AI receives structured typed DTOs, not flat key-value form blobs.
- Supplementary form engine (Phase 7) — was going to have the same limitation as competitors.
  **This gap is now solved by design** — see `semantic-form-engine-design.md`. Three-layer
  semantic form engine with `AiDescription` field metadata + `IFormSemanticContextBuilder`
  + ODK-inspired entity mapping. Novel in EHS SaaS space, research-backed.
- Competitors (Cority, VelocityEHS, Intelex) also have typed fields on their core incident
  forms — the differentiator is not just "we have dropdowns." It is state machine enforcement
  at domain layer, FK-constrained relationships, and MCP-first architecture.

---

## Issues 2–7 Still to Cover (1 by 1, honest verdict each)

These come from `ai-capabilities-research.md` Section 2:

**Issue 2 — Data Quality Is the Blocking Problem ✅ DISCUSSED**

Verdict: Genuine advantage on structural quality. Two accepted limitations.

What we solve:
- Typed enums + FluentValidation block garbage at the API boundary (422, not stored)
- CorrectiveAction.Status state machine enforces close-out discipline
- Single FK-linked domain model eliminates siloed data
- Semantic form engine enforces typed constraints even on custom fields
- Scoring on form submissions gives AI a summary signal without reading every field

Accepted limitations (not our problem to solve):

1. Near-miss under-reporting — workers don't report because of manager culture, fear of
   retaliation, office politics, and futility. This is a human/cultural problem that software
   cannot fundamentally fix. We can lower friction (Phase 12 mobile-first 3-tap reporting)
   and build trust via closed-loop notifications, but we cannot change workplace culture.
   This is an accepted data gap, not a platform failure.

2. Legacy data migration — new tenants have 5-15 years of history in CSVs, old systems,
   or paper. Strategy: onboard them first, get them using and loving the system, migrate
   historical data over time as a supported professional service (not a self-serve feature).
   AI-assisted migration (Claude reads their CSV + our domain model, generates mapping) makes
   this a hours-not-weeks effort per tenant. Charged as enterprise onboarding add-on.

**Issue 3 — Regulatory Language Is Not AI-Friendly Without Grounding ✅ DISCUSSED**

Verdict: We are the bridge between structured EHS data and regulatory documentation —
not the regulatory expert. Honest about the gap. Architecture is ready for when the gap closes.

What we DO NOT build:
- General-purpose regulatory Q&A chatbot → hallucination risk + legal liability → never

What we BUILD (Phase 17):
- Structured report pre-population: AI reads typed IncidentDto, pre-fills OSHA 300 /
  RIDDOR report template from system data. AI is the scribe; practitioner is the authority.
  Accurate because it comes from our typed data, not LLM knowledge. "Review before submission."
- Contextual regulatory links: when Incident.Type = LostTime, surface link to OSHA 300
  official guidance. Link to official source only — never interpret it.
- RegulatoryReference on form fields: already in FormFieldDescriptor (Phase 7).
  When compliance-critical field fails, show the regulation name and link. Not interpretation.
- "May need to report" flag: "Incidents of this type are typically reportable — verify
  with your EHS manager." Signal not advice. Human confirms.
- RegulatoryContent extensible table: we build the shelf. Enterprise customers with legal
  teams populate their jurisdiction's requirements. Industry partners provide content packs.

The architectural moat — forward-looking:
By the time Phase 17 is built, the regulatory content landscape will have evolved:
- OSHA eCFR has a public API today. UK legislation.gov.uk has a public API today.
- EU EUR-Lex is machine-readable today.
- Wolters Kluwer, Thomson Reuters — as MCP becomes enterprise standard (2026-2027),
  specialist regulatory content providers are likely to ship MCP servers exposing
  curated, maintained regulatory knowledge bases.
- When that happens: our RegulatoryContent hook + MCP architecture can consume it
  immediately. No architectural change needed. We built the shelf. We plug in the content
  when a trusted source makes it available at the right quality level.
- We do not have to be Wolters Kluwer. We have to be ready to connect to them.

**Issue 4 — Black Box AI Is Unacceptable to EHS Practitioners** ← NEXT

**Issue 3 — Regulatory Language Is Not AI-Friendly Without Grounding**
General LLMs hallucinate regulatory content (OSHA, RIDDOR, CDM). Wolters Kluwer Enablon
uses expert-curated knowledge bases. We do not have this. Honest gap to explore.

**Issue 4 — Black Box AI Is Unacceptable to EHS Practitioners**
Practitioners need to explain AI decisions to regulators and in legal proceedings.
Gartner: 60% of large enterprises adopting AI governance/explainability tools by 2026.
How does our audit trail (Phase 6) + AI service principal pattern address this?

**Issue 5 — Worker Surveillance Backlash**
46% of construction workers reject tracking devices. 59% reject biometric sensors.
Computer vision AI faces workforce trust and GDPR problems. How should we position?
(Hint: we are NOT building computer vision — the research says don't. But how do we
frame what we DO for worker safety vs. surveillance?)

**Issue 6 — No Vendor Has Solved ERP Integration at the Permit Level**
Prometheus Group (SAP SOLEX) is the closest but still point-to-point field mapping.
Our MCP + AI orchestrator pattern is the new approach. SAP and Dynamics 365 have shipped
MCP servers (2025-2026). This is the biggest potential moat. Discuss honestly:
what's real now vs. 18 months out?

**Issue 7 — Agentic AI Is Still Mostly Announcement, Not Production**
Benchmark Gensuite announced "first AI Agent framework for EHS" October 2025 — launch
announcement, not years of production data. Legal liability frameworks for AI agent actions
are unclear. Morgan Lewis (Feb 2026) recommends "AI outputs are advisory not determinative."
How does our architecture position us for when this matures?

---

## Key Decisions Already Made (Do Not Revisit)

- Typed domain model is the foundation — not a form engine. ADR locked.
- Supplementary form engine IS planned (Phase 7) but core entities stay typed. ADR-010.
- MCP Server planned for Phase 17. Architecture supports it cleanly.
- No autonomous permit approval. No autonomous severity changes. No general regulatory Q&A.
  See `ai-capabilities-research.md` Section 7 for full reasoning.
- No computer vision monitoring — out of scope, wrong domain, competitors have head start.
- `AiDescription` on every form field is the key design decision for Phase 7 AI-readability.

---

## The Developer Context

Parth Mashroo. 11+ years .NET backend. Building EHS Platform as a portfolio project.
Stack: .NET 8, Clean Architecture, CQRS + MediatR, EF Core 8, SQL Server 2022, JWT, Serilog.
All architecture decisions are locked — no re-debating tech choices.
He wants honest, transparent answers — no sugar-coating about what the platform already solves.

---

## How to Continue

Pick up from Issue 2. Go 1 by 1. For each issue:
1. State the issue clearly
2. Honest verdict: how does our architecture tackle it?
3. What's genuinely missing / what's a gap we haven't solved?
4. If there's a gap — propose a solution or accept the limitation

Do not discuss EHS-48 or any current code ticket — that is handled in a separate session.
