# ACGP Pilot Designs

The Agentic Cross-Enterprise Governance Pilot (ACGP) validates whether [AFIX](https://github.com/QBall-Inc/afix) governance metadata propagates correctly across enterprise boundaries using A2A and MCP protocols as they exist today.

**Status:** Design phase. Neither pilot has been executed. These documents specify how participating entities would validate AFIX governance propagation in realistic multi-party scenarios.

---

## Pilot Use Cases

### [Use Case 1: Research Workflow](acgp-research-pilot.md)

A cross-firm financial services research workflow. Three firms — sell-side orchestrator, data vendor, buy-side consumer — exercise a five-phase governance chain (Discovery, Negotiation, Execution, Audit, Failure) across both A2A and MCP transports. This is the pilot referenced in the [accompanying article](https://ashaykubal.com).

**What it validates:** AFIX governance propagation in the workflow pattern most common in capital markets today — delegated research across counterparties with auditable provenance at each firm boundary.

### [Use Case 2: Agentic Commerce](acgp-commerce-pilot.md)

A consumer purchase lifecycle. A consumer's AI agent discovers a product, negotiates terms, authorizes payment, executes the transaction, and handles a post-purchase dispute across five participating entities using both A2A and MCP transports. Exercises AP2 payment mandates, UCP checkout capabilities, and TAP identity verification alongside AFIX governance.

**What it validates:** AFIX governance propagation in a commerce context where AFIX coexists with AP2, UCP, and TAP on the same messages — proving that governance metadata and commerce protocol metadata operate independently without interference.

---

## Shared Design Principles

Both pilots share the same core validation criteria:

1. **Schema identity across transports.** The same AFIX component schema (identical SHA-256 hash) is used on both A2A and MCP. The schema is transport-invariant; only the binding differs.

2. **Cross-transport chain integrity.** A Governance Hop Record (GHR) chain that spans A2A and MCP maintains unbroken hash chain integrity and correct hop indexing.

3. **Entitlement enforcement, not just logging.** Out-of-scope actions are blocked at the protocol boundary (A2A `required: true` rejection, MCP gateway rejection), not merely logged after the fact.

4. **Independent audit reconstruction.** A Governance Observer (independent auditor) can reconstruct the complete governance state of any interaction from protocol logs alone, without access to any participant's internal systems.

---

## Relationship to the Article

Use Case 1 is abridged in the article's Section 6. The article presents the five-phase flow as a pilot design — what participating firms would execute to validate AFIX governance propagation. This repository contains the canonical, detailed version with full JSON payload examples at each phase.

Use Case 2 is not covered in the article. It extends AFIX validation to commerce scenarios where additional protocol layers (AP2, UCP, TAP) coexist with governance metadata.

---

## Contributing

Both pilot designs are open for review and contribution. If you are interested in participating as a pilot firm or have feedback on the design, open an issue or submit a pull request.
