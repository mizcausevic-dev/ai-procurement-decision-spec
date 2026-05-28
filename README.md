# AI Procurement Decision Card

[![Validate examples](https://github.com/mizcausevic-dev/ai-procurement-decision-spec/actions/workflows/validate.yml/badge.svg)](https://github.com/mizcausevic-dev/ai-procurement-decision-spec/actions/workflows/validate.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](https://opensource.org/licenses/MIT)

A draft open JSON specification for the **buyer-side artifact** in AI procurement. The first ten Kinetic Gain Protocol Suite specs cover vendor declarations (entity, prompts, agents, evidence, tools, AUP, tutor, disclosure, clinical, incident). The Decision Card closes the loop: it records the outcome of a buyer's review of one or more of those declarations.

Part of the **[Kinetic Gain Protocol Suite](https://suite.kineticgain.com/)**.

## Why this exists

The first ten Suite specs are all vendor-side. A school district that vets an EdTech AI tool, a hospital that evaluates a clinical AI tool, and a federal agency that adds a vendor to an Approved Vendor List currently have nowhere to publish their *review*. The decision exists — somewhere in a procurement office's filing cabinet — but it never enters the same machine-readable graph as the vendor's declaration.

The Decision Card fixes that. A buyer publishes a Decision Card at a well-known URL. The card cites the vendor documents that were reviewed (by URL and content hash), the criteria used, the outcome, any conditions attached, the appeals process, and (optionally) a cryptographic signature. The Suite's MCP tools and visualizer treat it as a first-class document — searchable, verifiable, cross-referenced against the vendor declarations it cites.

## Discriminator

Documents conforming to this spec carry the top-level field:

```json
{ "decision_card_version": "0.2", ... }
```

This matches the Suite's pattern (`aeo_version`, `clinical_ai_card_version`, etc.) for auto-detection across MCP tools, the GitHub Action, and the hosted validator. Values `"0.1"` and `"0.2"` are both valid — see the [v0.2 section](#v02-data-vault-targets) below.

### v0.2: data vault targets

v0.2 adds one optional top-level field — `data_vault_targets` — for declaring vendor-neutral PII tokenization vault targets the decision authorizes at runtime. The field is **additive and optional**: v0.1 documents stay valid against the v0.2 schema, and consumers that only understand v0.1 can ignore the new field safely.

```jsonc
{
  "decision_card_version": "0.2",
  // ... existing fields ...
  "data_vault_targets": [
    {
      "vendor": "skyflow",
      "vault_id": "v_xxxxxxxxxxxxxxxxxxxx",
      "vault_url": "https://acme-edu.vault.skyflowapis.com/",
      "fields_authorized": ["student.email", "student.parent_phone"],
      "reveal_roles": ["principal", "compliance-officer"],
      "reveal_audit_uri": "https://acme-edu.org/audit-stream",
      "notes": "EU-resident vault per GDPR carve-out in Condition C-2."
    }
  ]
}
```

**Vendor enum** (descriptive, not endorsement): `skyflow · piiano · nightfall · private-ai · very-good-security · evervault · custom · other`. The spec does not endorse a provider — the field is shape-only so downstream enforcement engines (`policy-as-code-engine`, `mcp-permission-broker`, runtime governance bridges) can route fields to the right vault and check whether a caller's role appears in `reveal_roles` before detokenizing.

Why this lives on the Decision Card: tokenization vendor + field list + reveal roles are **buyer-side configuration of the approval**, not a vendor declaration. The vendor's Tool Card or Clinical AI Card declares which PII the product collects; the Decision Card records which of those fields the buyer chose to vault and who may reveal them.

## Well-known URL convention

| Path | Purpose |
|---|---|
| `/.well-known/decisions/{decision_id}.json` | One Decision Card |
| `/.well-known/decisions/index.json` | Optional aggregator (array of Decision Card metadata, for crawlers) |

Buyers MAY publish per-decision files only, with no index. The index is a discoverability convenience.

## Document structure

```json
{
  "decision_card_version": "0.1",
  "decision_id": "SPRINGFIELD-DEC-2026-001",
  "issued_at": "2026-05-14T19:00:00Z",

  "buyer":          { ... who is the buyer ... },
  "decision_maker": { ... role of the deciding party, optional ... },
  "decision":       { "status": "approved-with-conditions", ... },
  "subject":        { ... what vendor + documents were reviewed ... },
  "criteria":       { ... policies and rubric used ... },
  "conditions":     [ ... attached conditions, if any ... ],
  "rationale":      "Free-form prose explaining the decision.",
  "history":        [ ... audit trail of state transitions ... ],
  "appeals":        { ... appeal deadline + process URL ... },
  "publication":    { ... publication URL + visibility ... },
  "signatures":     [ ... non-repudiation signatures ... ],
  "withdrawal":     { ... only present if status=withdrawn ... }
}
```

Six fields are required: `decision_card_version`, `decision_id`, `issued_at`, `buyer`, `decision`, `subject`, and `rationale`. Everything else is optional but recommended.

## Decision statuses

| Status | Meaning |
|---|---|
| `approved` | Use is approved, no conditions attached. |
| `approved-with-conditions` | Use is approved subject to the listed `conditions`. |
| `rejected` | Use is declined. No remediation pathway offered. |
| `rejected-with-remediation` | Use is declined; `conditions` describe what would change the answer. |
| `pending` | Review is in progress; no terminal decision yet. |
| `withdrawn` | A previous decision is revoked; `withdrawal` block explains. |
| `expired` | The decision's `effective_until` date has passed. |

## Example fixtures (`examples/`)

| File | Vertical | Version | Outcome |
|---|---|---|---|
| `district-edtech-approved-conditions.json` | EdTech (K-12 district) | v0.1 | Approved-with-conditions — 3 contractual/audit/technical conditions on an AI tutor |
| `hospital-clinical-rejected.json` | HealthTech | v0.1 | Rejected-with-remediation — FDA clearance scope mismatch + bias audit cohort gap |
| `federal-agency-approved.json` | Federal civilian | v0.1 | Approved for non-rights-impacting use per OMB M-24-10 + NIST AI RMF rubric |
| `district-edtech-vaulted-pii.json` | EdTech (K-12 district) | **v0.2** | Approved with `data_vault_targets` — Skyflow vault for student/parent PII, principal + compliance-officer reveal roles |

All validate against `decision-card.schema.json` (CI runs this on every push). The v0.2 example is parsed by the v0.2 schema only; older `kg-validate-action` releases (v0.1.1 and earlier) will reject the new field — bump to a v0.2-aware release when one ships.

## Composability with the rest of the Suite

A Decision Card references vendor documents via `subject.documents_reviewed[].url`. Suite tooling can:

- **Resolve** the referenced documents at their well-known URLs and re-validate them at decision time
- **Verify** the `content_hash` to detect document drift after the decision
- **Aggregate** multiple Decision Cards about the same vendor (e.g. "what % of K-12 districts approved AcmeTutor?")
- **Cross-check** an Decision Card's `criteria.rubric` against the buyer's own published AUP (via `criteria.policy_uris`)

## License

MIT-licensed. The specification text, JSON Schema, and example documents in this repository may be freely implemented, extended, redistributed, or incorporated into commercial or non-commercial products with attribution. Reference implementations of this spec (such as [mcp-kinetic-gain](https://github.com/mizcausevic-dev/mcp-kinetic-gain)) are licensed separately under AGPL-3.0.

## Kinetic Gain Protocol Suite

The eleventh spec in the family. The other ten cover vendor declarations:

| # | Spec | Vertical |
|---|---|---|
| 1 | AEO Protocol | Core |
| 2 | Prompt Provenance | Core |
| 3 | Agent Card | Core |
| 4 | AI Evidence Format | Core |
| 5 | MCP Tool Card | Core |
| 6 | AI Tutor Card | EdTech |
| 7 | Student AI Disclosure | EdTech (FERPA/COPPA) |
| 8 | Classroom AI AUP | EdTech |
| 9 | Clinical AI Disclosure | HealthTech (FDA SaMD + HIPAA) |
| 10 | AI Incident Card | Cross-cutting (EU AI Act Article 73) |
| **11** | **AI Procurement Decision Card** | **Cross-cutting (buyer-side)** |

- **Suite hub**: [suite.kineticgain.com](https://suite.kineticgain.com/)
- **Hosted validator**: [validator.kineticgain.com](https://validator.kineticgain.com/) — paste any Suite JSON, get inline validation
- **Unified MCP server + CLI**: [`mcp-kinetic-gain`](https://github.com/mizcausevic-dev/mcp-kinetic-gain) — every spec as an MCP tool
- **GitHub Action**: [`kg-validate-action`](https://github.com/mizcausevic-dev/kg-validate-action) — drop into any CI workflow
