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

**Issue 4 — Black Box AI Is Unacceptable to EHS Practitioners ✅ DISCUSSED**

Verdict: Non-issue for current architecture. Fully addressed for Phase 17.

Core posture: Our AI never makes decisions. It surfaces patterns from typed historical data.
All actions in the system are taken by named humans. The audit trail records who did what,
when, with what data visible to them. When an OSHA inspector asks "walk me through what
happened" — every action is attributable to a named human. There is no "AI decided X."

Phase 17 nuance: When AI suggestions are added (incident classification, CA recommendations),
the AI Service Principal pattern records BOTH the AI suggestion AND the human decision in
the same audit trail. Regulator can see: AI said High severity, named human overrode to
Medium, with timestamp and reason. That is explainable, defensible, auditable.

One-line verdict: We don't have a black box problem because our AI doesn't make decisions.
When Phase 17 adds suggestions, the audit trail records suggestion + human override.
Legally defensible by design. "AI outputs are advisory not determinative" — Morgan Lewis.

---

## AI Provider Strategy & Subscription Gating (Phase 17 Technical Design)

> Captured during Issue 4 discussion. Locked decisions for Phase 17 implementation.

### How AI Suggestions Work Technically

AI receives: typed data (IncidentDto, valid enum values as constraints) + domain prompt.
AI returns: JSON with suggested enum value + confidence score.
AI cannot suggest values outside our enums — hallucination is structurally limited.
Result is recorded via AI Service Principal before human sees it.

### Provider Landscape

| Provider | Models | Cost | Free Tier | Best For |
|---|---|---|---|---|
| Anthropic | Claude Haiku 4.5 / Sonnet 4.6 | $0.80–$15/MTok | None | Default. Best reasoning quality. Already in our ecosystem. |
| Groq | Llama 3.3 70B, Gemma, Mistral | $0.05–0.59/MTok | ✅ 14,400 req/day | Cheap tasks. Fastest inference available. |
| DeepSeek | V3, R1 | $0.14–0.55/MTok | ✅ Free quota | Very cheap. Strong reasoning for cost. |
| Mistral AI | Small / Medium | $0.10–0.30/MTok | ✅ Free tier | Cheap classification tasks. |
| Google Gemini | 2.0 Flash, Gemma 4 | Very cheap | ✅ 15 req/min free | Development + free-tier tenants. |
| Ollama | Llama, Mistral, Qwen (local) | $0 | ✅ Always free | Local development — zero API calls while coding. |

### "Proprietary AI" — The Honest Answer

No EHS vendor has trained their own model. Benchmark Gensuite "Genny AI", VelocityEHS
"VelocityAI", Cority "Applied AI" — all are foundation model APIs (OpenAI / Anthropic /
Azure OpenAI) with domain-specific prompt engineering and data context on top.

What IS proprietary and IS our IP:
- Prompt engineering for EHS domain tasks
- IFormSemanticContextBuilder output fed as context
- Constrained output (AI picks from our typed enums only)
- AI Service Principal audit recording pattern
- How suggestions surface to the user
- The integration layer connecting AI to the typed domain

Marketing language: "[Platform Name] AI" or "EHS Intelligence" — same as Salesforce
"Einstein AI." Never say "proprietary AI model" — inaccurate.

### Architecture — IAiSuggestionService Abstraction

Application layer defines the interface. Infrastructure layer implements per provider.
Swap provider via appsettings.json — no application code changes.

```csharp
// Application layer
public interface IAiSuggestionService
{
    Task<IncidentClassificationSuggestion> SuggestClassificationAsync(
        string description,
        IReadOnlyList<string> validTypes,
        IReadOnlyList<string> validSeverities,
        CancellationToken ct);
}

// Infrastructure — register one based on config
// ClaudeAiSuggestionService | GroqAiSuggestionService
// GeminiAiSuggestionService | OllamaAiSuggestionService
```

### Subscription Tier Gating

AI features are gated by tenant subscription tier. Free tenants get zero AI calls.
A single check in each AI handler: `if (!features.IsAiEnabled) return NotAvailable()`.
Per-tenant monthly quota tracked and enforced. No AI cost for non-paying tenants.

| Tier | AI Features | Quota | Model Tier |
|---|---|---|---|
| Free | None | 0 | — |
| Standard | Incident classification + CA suggestions | 100/month | Fast (Groq/Haiku) |
| Professional | + Pattern detection + report generation | 1,000/month | Full (Sonnet) |
| Enterprise | + Compliance agent + custom config | Unlimited | Configurable |

Provider routing by tier: Free tenants → no calls. Standard → Groq free tier.
Professional → Haiku. Enterprise → Sonnet. Costs scale with revenue.

Development strategy:
- Local dev: Ollama (zero cost, no API calls while coding)
- Testing: Gemini free tier or DeepSeek free quota
- Production: Groq (cheap) → Haiku (medium) → Sonnet (complex) by task

**Issue 5 — Worker Surveillance Backlash ✅ DISCUSSED**

Verdict: Not our jurisdiction. Platform neutral. One genuine feature worth building.

Platform neutrality: We provide tools. Client decides their rollout strategy, communication
approach, union consultation, pace of adoption. We do not mandate surveillance or prevent it.
Gradual rollout is standard ISO 45001 practice — workers adopt Phase 1 (frictionless reporting)
before Phase 2 (closed-loop notifications) before Phase 3 (deeper analytics). Our modular
per-tenant feature gating supports this naturally.

What we don't build: Computer vision, biometric monitoring, tracking devices.
If client has computer vision: they write to our MCP — we are the system of record.
Positioning: "We give workers a voice, not a camera pointed at them."

Just Culture language enforcement (Phase 17 — novel, build this):
Every piece of AI-generated text in the platform passes through a Just Culture system prompt
component. Just Culture (Sidney Dekker, aviation safety, ISO 45001) says accidents are system
failures not individual failures. Blame language suppresses reporting. System-focused language
encourages it. "PPE was not worn" not "worker forgot PPE." "Procedure gap identified" not
"employee violated rule." One system prompt addition. Every AI-generated CA description,
notification, pattern report, investigation template is Just Culture compliant by default.
No EHS competitor is doing this explicitly. Directly reduces under-reporting. Sales point.

**Issue 6 — No Vendor Has Solved ERP Integration at the Permit Level ✅ DISCUSSED**

Verdict: Tiered integration strategy. Tier 1 (screenshot AI) is the practical differentiator.
MCP is the long-term moat. Both are real — at different timelines.

Phase 13 — Work Permit Mobile Experience (standalone, no ERP needed):
- QR code per permit. Worker scans with phone (no app install — browser).
- Mobile page shows permit status (Active/Expired/Suspended) with controls required.
- Entry register: worker taps Enter/Exit → records identity, timestamp, geolocation.
- Supervisor verification: scan QR → see live entrant list.
- Offline support: queues scans, syncs on reconnect.
- Emergency muster: shows who is still on site right now.
- This alone exceeds most EHS platform permit experiences.

ERP integration tiers (offered as a menu — customer picks their tier):
- Tier 0: Manual ERP work order reference field. Zero effort. Always available.
- Tier 1: Screenshot/PDF upload + AI extraction. Operator uploads ERP work order
  screenshot or PDF. AI extracts work order number, asset, description, dates.
  Pre-fills permit. Operator confirms. Works with ANY ERP. No IT project. No consultant.
  ~85% extraction accuracy. THIS IS THE DIFFERENTIATOR — no competitor offers this.
- Tier 2: CSV/Excel import + AI column mapping. ERP team exports work orders weekly.
  AI maps columns to permit fields. Draft permits created automatically.
- Tier 3: iPaaS/middleware. We expose clean REST API + webhooks. Customer uses their
  existing Azure Logic Apps / MuleSoft / Boomi / n8n to connect SAP → our API.
  Zero effort on our side — just document the API properly.
- Tier 4: Direct ERP API adapter (SAP OData / Oracle REST). Professional services
  engagement, not a product feature. Charged separately. Per-ERP, per-version.
- Tier 5: MCP orchestration (Phase 17 + 12-18 month market maturity).
  SAP/Dynamics MCP servers ↔ AI Orchestrator ↔ our MCP server. Architecture ready.

Build plan: Phase 13 = Tier 0 + Tier 1 + mobile QR experience.
Phase 17 = Tier 2 + Tier 5 + REST API docs for Tier 3.
Tier 4 = on-demand professional services engagement.

**Issue 7 — Agentic AI Is Still Mostly Announcement, Not Production** ← NEXT

**Issue 7 — Agentic AI Is Still Mostly Announcement, Not Production ✅ DISCUSSED**

Verdict: Our architecture is more genuinely agentic than any current EHS launch announcement.
AI advises. Human decides. Always. This is the correct posture legally, ethically, and
commercially for a safety-critical domain.

Benchmark Gensuite's "first AI Agent framework for EHS" (Oct 2025) is a product announcement.
We built a foundation. Products get announced. Foundations compound.

What makes our architecture legitimately agentic:
- MCP server (Phase 17): AI-callable tools over full typed domain
- Outbox (Phase 9): reliable event-driven AI triggers — no polling
- AI Service Principal: every AI action attributed, timestamped, audited
- IAiSuggestionService: provider-agnostic, swappable, cost-optimised by tier
- Typed domain model: AI reasons precisely, not on form blobs
- Semantic form engine: even custom forms are AI-readable
- Just Culture language enforcement: AI-generated text follows safety communication standards

What our AI does (agentic read + detect + notify + suggest):
✅ Reads data on demand via MCP tools
✅ Detects conditions from Outbox events (CA overdue, permit expiring, silence anomaly)
✅ Sends notifications attributed to AI Service Principal — fully audited
✅ Generates reports, summaries, pre-filled regulatory documents
✅ Suggests classifications — human confirms, audit records both

What our AI never does (legal documents, determinative decisions):
❌ Approves permits | ❌ Signs JSAs | ❌ Creates CAs autonomously
❌ Changes severity without confirmation | ❌ Answers regulatory questions

Liability posture (Morgan Lewis Feb 2026): "AI outputs are advisory not determinative."
Every AI action is attributed, timestamped, paired with the human decision that followed.
If a regulator asks "what did your AI do" — we have a complete immutable answer.
That is not possible on platforms where AI writes to form fields with no audit trail.

---

## ALL 7 ISSUES COMPLETE ✅

This discussion is fully documented. All verdicts locked.
See individual issue entries above for full detail.
Next: return to implementation (EHS-42 and Phase 4 completion).

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
