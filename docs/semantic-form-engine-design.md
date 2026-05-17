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
