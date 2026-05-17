# Semantic Form Engine Design — Phase 7

> Written May 2026. Captures the architectural decision for how Phase 7's dynamic form engine
> remains AI-readable without sacrificing flexibility.
>
> Problem identified during EHS-48 session. Research-backed. No prior published solution exists
> specifically in EHS SaaS — this is novel design territory.

---

## The Problem

Our Phase 7 form engine allows tenant administrators to create custom forms — contractor safety
inspections, site audits, permit checklists, custom incident supplements. These are dynamic:
field keys are generated, labels are set by the tenant, structure varies per form.

The AI limitation: a form submission stored as `{ "field_23": "yes", "field_7": 3 }` is
completely opaque to an LLM without additional context. The AI cannot reason about what the
fields mean, what valid values are, or what a "yes" vs "no" implies for compliance.

This is identical to the limitation that affects all form-engine-based EHS platforms
(SafetyCulture, older Intelex modules, Cority's configurable modules). The gap between our
typed core entities and our supplementary forms would otherwise be a real AI disadvantage.

**The solution: a three-layer semantic form engine.** Research-backed (ODK XForms Entities
layer, FMEA ontology papers 2024-2025, MetaConfigurator research, OG-RAG EMNLP 2025).

---

## The Non-Negotiable Architectural Rule

> **A form submission must NEVER be stored without a versioned pointer to the template
> that generated it.**

If this link is broken or missing, AI interpretability is permanently lost for those submissions.
Every submission record carries `FormTemplateId` and `FormTemplateVersion`. This is not optional.

---

## Three-Layer Architecture

### Layer 1 — Form Template with AI Metadata

Every field in a form template carries AI-specific metadata beyond the display label.
The key addition is `AiDescription` — a machine-oriented explanation written for LLM consumption,
not for the user who fills the form.

```csharp
public class FormTemplate : BaseEntity, ITenantEntity
{
    public Guid TenantId { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Version { get; set; } = string.Empty;
    public string? EntityType { get; set; }  // e.g. "ContractorSafetyInspection"
    public List<FormFieldDescriptor> Fields { get; set; } = new();
}

public class FormFieldDescriptor
{
    public string FieldKey { get; set; } = string.Empty;
    public string DisplayLabel { get; set; } = string.Empty;

    // THE KEY ADDITION — written for LLM, not for the user
    public string AiDescription { get; set; } = string.Empty;

    public string DataType { get; set; } = string.Empty;  // "boolean", "date", "string", "enum", "number"
    public string? SemanticConcept { get; set; }           // "ContractorCompliance.InductionStatus"
    public string? RegulatoryReference { get; set; }       // "HSE CDM 2015 Regulation 12"
    public bool IsComplianceCritical { get; set; }
    public List<string>? AllowedValues { get; set; }       // For enum fields — human-readable strings
    public string? Unit { get; set; }                      // "days", "metres", "kg"

    // Entity mapping (ODK-inspired) — maps this field to a typed domain entity property
    public string? EntityPropertyPath { get; set; }        // "Contractor.HasValidInduction"
}
```

EF Core 8 JSON column stores `List<FormFieldDescriptor>` as a JSON column on `FormTemplates`.
No normalized field-per-row table needed. One `HasJsonConversion()` call in the EF config.

**Example field definition:**
```json
{
  "fieldKey": "hasValidInduction",
  "displayLabel": "Contractor has valid induction certificate?",
  "aiDescription": "Confirms contractor completed mandatory site induction before starting work. True = compliant. False = non-compliant — contractor must not begin work until induction is completed.",
  "dataType": "boolean",
  "semanticConcept": "ContractorCompliance.InductionStatus",
  "regulatoryReference": "HSE CDM 2015 Regulation 12",
  "isComplianceCritical": true,
  "allowedValues": null,
  "entityPropertyPath": "Contractor.HasValidInduction"
}
```

**What `AiDescription` should contain (rules for template authors):**
- What the field measures (not what the label says — what it actually means)
- What each value implies in the compliance or safety context
- What action is triggered by a non-compliant response
- Never use jargon that is specific to your UI ("see section 2 dropdown")

### Layer 2 — Submission Envelope

```csharp
public class FormSubmission : BaseEntity, ITenantEntity
{
    public Guid TenantId { get; set; }
    public Guid FormTemplateId { get; set; }
    public string FormTemplateVersion { get; set; } = string.Empty;  // Pinned at submission time
    public Guid? ParentEntityId { get; set; }       // e.g. the IncidentId this supplements
    public string? ParentEntityType { get; set; }   // "Incident", "WorkPermit", "Site"
    public Guid SubmittedById { get; set; }
    public Dictionary<string, object?> Values { get; set; } = new();
}
```

The `ParentEntityId` + `ParentEntityType` link is how the form data anchors to the typed domain.
An AI reading an Incident can discover all supplementary form submissions linked to it and
retrieve their semantic context. The typed entity is always the anchor.

### Layer 3 — The Semantic Context Assembler

An Application-layer service that an AI feature calls before sending form data to an LLM.
It joins submission values to their field descriptors and renders a structured context document.

```csharp
public interface IFormSemanticContextBuilder
{
    Task<SemanticFormContext> BuildAsync(Guid submissionId, CancellationToken ct);
}

public class SemanticFormContext
{
    public string FormName { get; init; } = string.Empty;
    public string FormVersion { get; init; } = string.Empty;
    public string? EntityType { get; init; }
    public List<SemanticFieldContext> Fields { get; init; } = new();
    public string ToAiContextString() { /* renders the structured text block below */ }
}
```

**Output of `ToAiContextString()` for an LLM:**

```
FORM: Contractor Safety Inspection v2.1
SUBMITTED: 2026-05-17 09:30 UTC by John Smith
LINKED TO: Site Inspection #INS-2847 at Site B

FIELDS:
─────────────────────────────────────────
hasValidInduction [ContractorCompliance.InductionStatus | HSE CDM 2015 Reg 12 | ⚠ COMPLIANCE CRITICAL]
  "Confirms contractor completed mandatory site induction before starting work.
   True = compliant. False = non-compliant — work must not proceed."
  Response: TRUE ✓ COMPLIANT

lastInductionDate [ContractorCompliance.InductionDate]
  "Date of most recent induction completion. Must be within 12 months."
  Response: 2026-03-01 (77 days ago — within 12-month valid window) ✓

inductionTrainer [ContractorCompliance.InductionTrainer]
  "Name of the certified trainer who delivered the induction."
  Response: "Sarah Chen (EHS Officer)"
─────────────────────────────────────────
COMPLIANCE SUMMARY: 3/3 critical fields compliant. 0 non-compliant responses.
```

The LLM never sees `field_23: "yes"`. It sees semantic context. The `IFormSemanticContextBuilder`
is a pure Application layer service — no HTTP, no LLM calls, fully unit testable.

---

## Entity Mapping — The Bridge to Typed Domain

Inspired by ODK XForms Entities layer (production-proven in WHO/UNICEF field deployments
across 100+ countries). When a form field declares `EntityPropertyPath`, that field's value
can be written back to the typed domain entity, not just stored as a form blob.

```csharp
// After form submission, the entity mapper runs:
// FormSubmission.Values["hasValidInduction"] = true
// → Contractor.HasValidInduction = true
// This is the typed entity, not the form blob
```

This means:
- The MCP tool `get_contractor` returns the typed `ContractorDto` including `HasValidInduction: true`
- The AI does not need to call `get_form_submission` to discover the compliance status
- The form is the collection mechanism; the typed entity is the system of record

Not every field needs an entity mapping. Supplementary context fields (notes, photos, open text)
stay as form data. Compliance-critical fields that belong in the typed domain are mapped.
The split is a deliberate design choice per form template, made by the template author.

---

## Why Not a Full OWL Ontology

The research found the academically correct approach is an OWL/RDF ontology with SPARQL queries
over a knowledge graph (OSHDO-Core, Construction Safety Ontology, semantic_forms). However:

- OWL/RDF/SPARQL is a foreign toolchain to a .NET shop
- OSHDO-Core is research-grade, not production tooling
- Full ontology maintenance is a sustained content operations problem, not a software problem
- OG-RAG (EMNLP 2025, Microsoft Research) achieves +55% recall and +40% correctness vs.
  standard RAG using ontology-grounded hyperedges — but requires no RDF toolchain if the
  semantic concepts are expressed as structured metadata rather than URI-dereferenced triples

**Our `SemanticConcept` field (`"ContractorCompliance.InductionStatus"`) gives us ~80% of what
a full OWL URI gives us, at 5% of the complexity.** The LLM understands dot-notation namespaced
concepts without dereferencing a URI. Full OWL is a future optimisation if volume justifies it.

---

## Implementation Path

Phase 7 builds the form engine. The semantic layer is designed in from day one — not retrofitted.

| Step | What | Effort |
|---|---|---|
| `FormTemplate` entity + EF config + JSON column for `Fields` | Phase 7 | Small |
| `FormSubmission` entity + EF config + `ParentEntityId` link | Phase 7 | Small |
| `IFormSemanticContextBuilder` + `SemanticFormContext` | Phase 7 | Medium |
| Template authoring for first 3 form types with full `AiDescription` | Phase 7 | Medium |
| Entity mapper (form field → typed entity property) | Phase 7 | Medium |
| AI features that consume `IFormSemanticContextBuilder` | Phase 17 | Follows naturally |

The hard cost is **human knowledge work**: writing `AiDescription` for each field on each form
template. 10 form types × 20 fields = 200 descriptors. That requires an EHS domain expert,
not a developer. Plan for it explicitly.

---

## What This Means for Our AI Differentiation

With this design:

1. Our supplementary forms are AI-readable — the same limitation that affects SafetyCulture,
   Intelex, and Cority's configurable modules does not apply to us.
2. The typed domain model advantage extends into the form layer — form data anchors to typed
   entities, not floats as a disconnected blob.
3. The MCP server exposes typed entities. Forms are surfaced as semantic context on demand.
   The AI always reasons about the typed model, enriched by form data when relevant.
4. This is the only known architecture in EHS SaaS that solves both flexibility
   (tenant-configurable forms) and AI interpretability. The research found no published
   equivalent in this domain.

---

## Prior Art Referenced

- **ODK XForms Entities Specification** — entity mapping concept (WHO/UNICEF field deployments)
- **MetaConfigurator** (Stuttgart University, 2024) — JSON Schema + LLM bidirectional form tool
- **FMEA Ontology + LLM papers** (ArXiv 2511.17743, 2510.15428, Cambridge Core 2025)
- **OG-RAG** (EMNLP 2025, Microsoft Research) — ontology-grounded RAG, +55% recall
- **OSHDO-Core** (Liverpool University) — occupational safety OWL ontology (context only)
- **Form.io JSON Schema** — schema/submission separation pattern
- **Semantic Layer pattern** (Cube.dev, dbt MetricFlow, OSI standard Sep 2025)

---

*Designed May 2026. Implement in Phase 7. The AI features that consume this (Phase 17)
depend on this foundation being in place.*

---

## Mental Model — Three Types of Fields

> This section exists because the distinction between typed fields, mappable fields, and
> purely custom fields is the most commonly misunderstood part of this design.
> Read this before touching any implementation.

### The Core Rule

The form engine and the typed domain model are **two different things that work together**.
The typed domain model never changes because a tenant builds a form. A tenant cannot
redesign `Incident` or add a field to `Contractor` through the form engine. Those are
locked. The form engine is for **supplementary data** collected on top of core entities.

### TYPE 1 — Fixed Typed Fields (Developer only, never in the form engine)

These are C# entity properties, EF Core columns, enforced by the domain layer.

```
Incident.Severity    Incident.Status      Incident.Type
CorrectiveAction.DueDate                  CorrectiveAction.Status
WorkPermit.Type      Contractor.Name      Contractor.TenantId
```

- Developer defines in C# entities, EF Core migrates to SQL
- AI reads directly from typed DTOs via MCP — no form lookup needed
- Tenant admin has no access to these

### TYPE 2 — Mappable Form Fields (Developer pre-registers; rare)

A small set of fields (~10–20 across the whole platform) where data is collected via a
form but is also important enough to live on the typed entity — either because domain
business logic depends on it, or because the AI should be able to read it directly without
digging through form submissions.

The developer registers these in `EntityPropertyRegistry`:
```csharp
["Contractor.HasValidInduction"] = (entity, val) =>
    ((Contractor)entity).HasValidInduction = (bool?)val,
```

When a tenant admin builds a form, they can optionally pick from this pre-registered list.
The value is then written to BOTH: the form JSON blob AND the typed entity property.

**Decision rule for whether something belongs here:**
- Is there domain business logic that depends on this value? → Typed property (Type 1 or Type 2)
- Is it universally important enough that the AI should read it without a form lookup? → Type 2
- Is it supplementary, tenant-specific, or reporting-only? → Type 3 (never needs mapping)

### TYPE 3 — Purely Custom Form Fields (Tenant admin invents freely)

Anything the tenant admin wants to collect. `CASignature`, `WeatherConditions`,
`WitnessName`, `EquipmentSerialNumber`, regulatory checklist items, open text, photos.
No developer involvement. No typed property.

- Stored as JSON blob in `FormSubmission.Values`
- AI reads via `IFormSemanticContextBuilder` → semantic context string
- `AiDescription` written by the tenant admin makes the field meaningful to the AI
- Nothing breaks when a field has no `EntityPropertyPath` — it simply stores as form data

### The `CASignature` Example

```
Tenant admin creates "CASignature" field
         │
Does a typed property exist?  NO
         │
Does anything break?          NO — stores as form blob
         │
Can the AI read it?           YES — via IFormSemanticContextBuilder
         │
Developer involvement needed? NONE
```

### Full Lifecycle — Who Does What

```
DEVELOPER (once, at build time)
  │  Creates typed entities (Incident, Contractor, WorkPermit...)
  │  Registers ~10-20 mappable paths in EntityPropertyRegistry
  ▼

TENANT ADMIN (whenever a new form is needed)
  │  Opens form builder
  │  Adds fields freely — whatever the business needs
  │  For each field: writes AiDescription, sets compliance flag
  │  Optionally selects EntityPropertyPath from developer's registered list
  │  Fields without a path? Fine — they store as form data
  ▼

FIELD WORKER (daily)
  │  Opens rendered form (RJSF renders from schema)
  │  Fills in fields including any custom ones
  │  Submits
  ▼

SYSTEM (on submission)
  │  Stores ALL fields as FormSubmission JSON blob
  │  For fields WITH EntityPropertyPath → also writes to typed entity
  │  For fields WITHOUT → stored in blob only (no error, no gap)
  ▼

AI (on demand via MCP)
     Reads typed entity directly (fast, typed, precise)
  +  Calls IFormSemanticContextBuilder for supplementary context
  =  Full picture: typed domain + semantic form data
```

---

## Pre-Built Industry Form Templates

> Design decision: May 2026. Implement in Phase 7 alongside the form engine.

### The Idea

The platform ships a library of **system form templates** — pre-built, pre-described,
organised by industry and regulatory standard. Tenants can use them as-is or clone and
customise. All `AiDescription`, `SemanticConcept`, and `RegulatoryReference` fields are
pre-populated by the developer.

This solves three problems at once:
1. Tenants get instant value on day one — no blank-canvas problem
2. The `AiDescription` authoring burden is handled by the developer for common forms
3. Regulatory standard forms become a moat — no competitor ships semantically-enriched
   ISO/OSHA/CDM templates out of the box

### Entity Design

```csharp
public class FormTemplate : BaseEntity, ITenantEntity
{
    public Guid TenantId { get; set; }        // Null for system templates
    public bool IsSystemTemplate { get; set; } // System templates: read-only, not deletable
    public string? IndustryTag { get; set; }   // "Construction", "OilAndGas", "Mining", "Generic"
    public string? StandardReference { get; set; } // "ISO 45001:2018", "OSHA 1904", "UK CDM 2015"
    public Guid? ClonedFromTemplateId { get; set; } // Audit trail of which system template was cloned
    // ... existing fields
}
```

### Tenant Workflow

```
Tenant Admin opens Form Library
         │
         ├── My Templates (tenant-created)
         │
         └── System Templates (platform-provided, read-only)
                  │
                  ├── [ISO 45001] Hazard Identification Checklist
                  ├── [OSHA 300]  Incident Investigation Supplement
                  ├── [UK CDM 2015] Contractor Safety Induction
                  ├── [Construction] Pre-Task Safety Briefing
                  ├── [Oil & Gas]  Permit to Work Checklist
                  └── ...
                  │
                  Use as-is ──► rendered directly, all AI descriptions pre-written
                  OR
                  Clone ──────► creates a copy in My Templates → customise freely
```

### Phase 7 Starter Pack (first-party templates to ship)

| Template | Standard | Industry |
|---|---|---|
| Contractor Safety Induction | UK CDM 2015 / ISO 45001 | Generic |
| Incident Investigation Supplement | OSHA 1904 / RIDDOR 2013 | Generic |
| Hazard Identification Checklist | ISO 45001:2018 Cl. 6.1 | Generic |
| Pre-Task Safety Briefing (JSA/JHA) | OSHA General Industry | Construction |
| Hot Work Permit Checklist | NFPA 51B | Construction / Oil & Gas |
| Confined Space Entry Checklist | OSHA 1910.146 | Manufacturing / Oil & Gas |
| LOTO Pre-Task Verification | OSHA 1910.147 | Manufacturing |
| Environmental Spill Response | EPA / ISO 14001 | Generic |

Each template: all `AiDescription` fields written, `RegulatoryReference` populated,
`IsComplianceCritical` flags set. Ready for AI reasoning on day one.

---

## Form Builder Library Selection

> Researched May 2026. Decision locked for Phase 7.

### Decision

**RJSF (`@rjsf/core`, Apache 2.0) as the renderer + a custom lightweight admin builder page.**

SurveyJS renderer (MIT, never the creator package) is a valid fallback if RJSF proves insufficient for rendering complexity. The builder is a custom admin UI regardless.

---

### Library Comparison

| Library | Licence | Builder UI | Renderer | Custom Metadata | GitHub Health | Verdict |
|---|---|---|---|---|---|---|
| **RJSF** | Apache 2.0 ✓ | Community add-on only (low activity) | Excellent — MUI/Bootstrap/shadcn themes | Arbitrary keys pass through silently. No registration needed. | 15.8k stars, 358 contributors, active 2025 | **Recommended** |
| **JSON Forms** | MIT ✓ | Research/PoC only — not production | Very good — React/Angular/Vue | `x-` prefixed extensions documented | 2.6k stars, active 2025 | Runner-up |
| **Formily** | MIT ✓ | `@formily/designable` (Ant Design-coupled, slowing) | Excellent reactive forms | `x-*` extensions first-class in schema | 12.7k stars, docs mostly Chinese | Conditional — wrong stack fit |
| **SurveyJS renderer** | MIT ✓ | **Creator is COMMERCIAL ~$573/dev** | Best-in-class, lazy load, React/Vue | `Survey.Serializer.addProperty()` — works cleanly | Renderer 4.3k stars, active | Fallback renderer only |
| **SurveyJS creator** | **Commercial ✗** | Polished, best drag-drop | N/A | Same as renderer | 1.3k stars | **Never use without licence** |
| **Form.io** | OSL-3.0 ✗ | Full stack Node.js | Good | Proprietary schema | — | **Hard disqualified** — OSL-3.0 copyleft covers SaaS deployment |
| **RHF + Zod + custom** | MIT ✓ | Would have to build from scratch | Would have to build from scratch | Full control | 42k stars (RHF alone) | Last resort only |

---

### Why RJSF

1. **Licence is unambiguous.** Apache 2.0 — commercial SaaS use is explicitly permitted. No surprise commercial tiers.
2. **Custom metadata requires zero ceremony.** Our `aiDescription`, `semanticConcept`, `regulatoryReference`, `isComplianceCritical`, `entityPropertyPath` fields live directly on each field's JSON Schema entry. RJSF passes unknown keys through to custom widgets — no registration, no schema extensions.
3. **Native .NET backend fit.** JSON Schema Draft 7 is natively supported by `NJsonSchema` and `System.Text.Json` in .NET 8. Schema stored as EF Core 8 JSON column, served via REST, consumed directly by `<Form schema={...} />`. No translation layer.
4. **Builder gap is not a problem here.** The EHS admin template builder is a structured authoring UI for trained super-admins, not a public drag-drop canvas. It needs: add field (type picker), fill AI metadata, reorder/delete, real-time preview. That is 1–2 sprint tickets using `<Form schema={...} />` as the preview. No off-the-shelf builder needed.
5. **Ecosystem size.** 15.8k stars, 358 contributors, MUI/Bootstrap/shadcn/Ant Design theme packages maintained separately. Bug reports get responses.

---

### SurveyJS — The Licence Trap to Avoid

SurveyJS markets itself as open source. This is misleading:

- `survey-library` (renderer): **MIT. Genuinely free.**
- `survey-creator` (builder): **Commercial licence. ~$573 USD per developer per year (Basic plan).**

If a future engineer installs `survey-creator` without reading the licence, the project has a commercial licence violation. The renderer is safe to use as a fallback for rendering quality. The creator must never enter the codebase.

---

### Integration Pattern with Our Semantic Layer

RJSF's JSON Schema for a single field:

```json
{
  "type": "boolean",
  "title": "Contractor has valid induction certificate?",
  "aiDescription": "Confirms contractor completed mandatory site induction before starting work. True = compliant. False = non-compliant — contractor must not begin work until induction is completed.",
  "semanticConcept": "ContractorCompliance.InductionStatus",
  "regulatoryReference": "HSE CDM 2015 Regulation 12",
  "isComplianceCritical": true,
  "entityPropertyPath": "Contractor.HasValidInduction"
}
```

RJSF passes `aiDescription`, `semanticConcept`, `regulatoryReference`, `isComplianceCritical`, and `entityPropertyPath` through to our custom widgets and to the `IFormSemanticContextBuilder` without touching them. The backend stores the full schema (custom fields included) as an EF Core 8 JSON column. Round-trip is lossless.

---

### Bundle Size

- RJSF core: ~150–175 KB gzipped
- With MUI theme: adds ~32% overhead
- Acceptable for enterprise B2B — not a consumer app where first-paint matters

---

*Researched May 2026. Supersedes any prior assumption about SurveyJS being the default choice.*
