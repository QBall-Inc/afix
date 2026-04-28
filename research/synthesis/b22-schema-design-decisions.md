# Synthesis: Schema Design Decisions (B.2.2)
_Date: 2026-04-20 | Status: Final_

---

> ## CRITICAL FRAMING NOTE — READ BEFORE ANYTHING ELSE
>
> This research contains a mix of **established facts** and **original proposals**. Every claim in the article must be clearly framed as one or the other. The table below is the authoritative reference for what exists vs. what is being proposed. All `finos.org/afix/` URIs, schema `$id` references, and CEAG working group references are **illustrative** — they represent what we propose, not what exists today. When drafting the article, use the indicative mood ("X exists") only for items in the left column and the subjunctive/propositional mood ("X would…", "we propose…") for items in the right column.
>
> ### What Exists vs. What We Propose
>
> | Status | Category | Item | Evidence Level |
> |--------|----------|------|----------------|
> | **EXISTS** | Protocol Stack | A2A (Google) — agent-to-agent coordination protocol | T1: Published spec, open-source |
> | **EXISTS** | Protocol Stack | MCP (Anthropic) — model context protocol for tool invocation | T1: Published spec, open-source |
> | **EXISTS** | Protocol Stack | AP2 (Google) — cryptographic payment mandates (Intent, Cart, Payment) | T1: Published spec, 60+ partners |
> | **EXISTS** | Protocol Stack | UCP (Google + Shopify) — universal commerce protocol with REST/MCP/A2A bindings | T1: Published spec |
> | **EXISTS** | Protocol Stack | TAP (Visa + Cloudflare) — agent identity verification via RFC 9421 signatures | T1: Published spec, open-source |
> | **EXISTS** | Market Evidence | Bloomberg built proprietary governance middleware on MCP | T1: Bloomberg official announcement |
> | **EXISTS** | Market Evidence | FactSet built proprietary governance middleware on MCP | T1: FactSet official announcement |
> | **EXISTS** | Market Evidence | LSEG built proprietary governance middleware on MCP (Trusted AI Ready Content) | T1: LSEG official announcement |
> | **EXISTS** | Market Evidence | All three built identical categories: identity, access control, audit logging, rate limiting | T2: Analyst synthesis of T1 sources |
> | **EXISTS** | Protocol Mechanism | A2A typed extension system with `required` flag and `extensions[]` arrays | T1: A2A spec |
> | **EXISTS** | Protocol Mechanism | MCP `_meta` field (untyped, any JSON, no schema validation) | T1: MCP spec |
> | **EXISTS** | Protocol Mechanism | Kong MCP Registry with gateway enforcement (ACLs, rate limiting) | T1: Kong announcement |
> | **EXISTS** | Protocol Mechanism | UCP Capability/Extension/Profile pattern with multi-transport bindings | T1: UCP spec |
> | **EXISTS** | Protocol Mechanism | AP2 mandate architecture (SD-JWT-VC format, cryptographic signing) | T1: AP2 spec |
> | **EXISTS** | Historical Precedent | FIX Protocol — governance metadata embedded in trade messages (1993) | T1: FIX spec |
> | **EXISTS** | Historical Precedent | HL7 FHIR — capability-based design, multi-transport binding, extension mechanism (2011) | T1: HL7 spec |
> | **EXISTS** | Protocol Mechanism | MCP SEP-2133 (formal extension framework) — Final status, shipped | T1: MCP spec, verified April 2026 |
> | | | | |
> | **WE PROPOSE** | Governance Schema | AFIX — protocol-agnostic Agentic Financial Interoperability eXchange | Original contribution |
> | **WE PROPOSE** | Schema Component | FS Agent Cards (organizational identity, LEI, jurisdiction, escalation) | Our design, informed by existing patterns |
> | **WE PROPOSE** | Schema Component | Compliance Events (six-field universal audit primitive) | Our design; six-field convergence from FIX/FHIR/RIXML is our analysis |
> | **WE PROPOSE** | Schema Component | Entitlement Interchange (cross-enterprise data/service access rights) | Our design |
> | **WE PROPOSE** | Schema Component | Liability Metadata (multi-party liability allocation, EMV-inspired) | Our design; EMV analogy is our framing |
> | **WE PROPOSE** | Schema Component | Governance Hop Record / GHR (multi-hop chain integrity) | Our design; SWIFT UETR analogy is our framing |
> | **WE PROPOSE** | Binding Spec | A2A binding — five AFIX extensions registered in `extensions[]` | Our design using existing A2A extension mechanism |
> | **WE PROPOSE** | Binding Spec | MCP binding — `_meta` conventions with `finos.org/afix/` prefix | Our design using existing MCP `_meta` field |
> | **WE PROPOSE** | Binding Spec | Gateway enforcement patterns (Kong YAML, Apigee XML, custom proxy) | Our design using existing gateway products |
> | **WE PROPOSE** | Namespace | `https://finos.org/afix/v1/` — all schema `$id` URIs | Illustrative; no such namespace exists today |
> | **WE PROPOSE** | Architecture | Four-layer protocol stack (Transport → Commerce → Payment Auth → Identity → Governance) | Our analytical framework; layer separation is our original contribution |
> | **WE PROPOSE** | Architecture | Asymmetric binding model (A2A protocol-native vs. MCP gateway-mediated) | Our original finding and framing |
> | **WE PROPOSE** | Architecture | Capability/Extension/Profile pattern for AFIX (modeled on UCP) | Our design adapting UCP's proven pattern |
> | **WE PROPOSE** | Pilot | ACGP (Agentic Commerce Governance Pilot) — seven-step dual-transport test | Our design; has not been run |
> | **WE PROPOSE** | Institution | CEAG (Cross-Enterprise Agent Governance) working group at FINOS | Our proposal; no such working group exists |
> | **WE PROPOSE** | Institution | FINOS Community Specification for AFIX | Our proposal; FINOS has not adopted this |
> | **WE PROPOSE** | Analogy | Bloomberg/FactSet/LSEG as "pre-HL7v2 hospitals" | Our framing; the triple-build is fact, the analogy is ours |
> | **WE PROPOSE** | Analogy | FIX → A2A binding pattern, FHIR → Capability/Extension/Profile pattern | Our framing; the protocols are fact, the mapping is ours |
> | **WE PROPOSE** | Design Decision | `ap2_mandate_ref` in `ext/agentic-commerce` domain extension (moved from base schema — Phase A correction for protocol-agnosticism) | Our design decision |
> | **WE PROPOSE** | Design Decision | EMV-inspired proportional liability via `governance_certification_level` | Our design; EMV liability shift is fact, application to agent governance is ours |

---

## Purpose

This document synthesizes two workstreams: AFIX Binding Specifications and ACGP Pilot Update & Production Decisions. Together they move AFIX from thesis to implementable design — concrete JSON schemas, transport-specific binding specs, a seven-step dual-transport pilot, and article production decisions.

---

## Thesis Status: Design-Validated

The protocol surface research confirmed the thesis with market evidence. This workstream validates it with engineering: the four AFIX schema components bind to both A2A and MCP transports without modification. The schema is genuinely protocol-agnostic. The enforcement mechanisms are genuinely different. Both work.

---

## Key Finding 1: Four Concrete Schema Components + GHR

AFIX comprises four governance schema components plus the Governance Hop Record (GHR) for multi-hop chains. Each is designed as a standalone JSON Schema. The `$id` URIs shown below (e.g., `https://finos.org/afix/v1/`) are **illustrative** — they represent the proposed namespace if FINOS adopts AFIX as a Community Specification. No such namespace exists today; this is our proposal for what it would look like:

| Component | Purpose | Key Fields | What It Answers |
|-----------|---------|------------|-----------------|
| **FS Agent Card** | Organizational identity & governance posture | `organization.lei`, `regulatory_jurisdiction`, `authorized_actions.authority_thresholds`, `escalation_chain`, `interop_declarations` | "Who is this agent, what can it do, and who do we call when it can't?" |
| **Compliance Events** | Universal audit primitive | Six-field interop core: `actor`, `target`, `action`, `timestamp`, `authorization`, `outcome`. Regulatory compliance extension adds 9 fields (`model_id`, `dataset_version`, microsecond timestamps, `log_integrity_hash`, etc.). Domain extensions via `authorization._ext_refs`. | "What governance-relevant action occurred, and was it authorized?" |
| **Entitlement Interchange** | Cross-enterprise data/service access rights | `grant_type`, `scope`, `data_classification`, `redistribution` enum (`PROHIBITED`/`DISPLAY_ONLY`/`DERIVED_ONLY`/`FULL`) | "What is this agent entitled to access, and can it redistribute?" |
| **Liability Metadata** | Multi-party liability allocation | `liability_model` (5 values), `parties[]` with roles and `governance_certification_level`. Phased adjudication: ISDA-analog bilateral (Phase 1) → Basel peer-monitoring (Phase 2). EMV proportional shift mechanism for `PROPORTIONAL_BY_CERTIFICATION` rule. | "Who is responsible when something goes wrong, and in what proportion?" |
| **GHR** | Multi-hop chain integrity | `chain_id` (SWIFT UETR analog), `hop_index`, `parent_hash` (SHA-256), `compensation` definition | "Can we trace the full governance chain across multiple hops and transports?" |

**Design contributions:**
- `redistribution` enum in Entitlement Interchange — directly addresses the Bloomberg/FactSet/LSEG data licensing problem
- `ap2_mandate_ref` moved to `ext/agentic-commerce` domain extension (Phase A stress-test: base schema must be protocol-agnostic). Cross-protocol traceability via `authorization._ext_refs`
- Two-tier Compliance Event structure: interop core (6 fields) + regulatory compliance extension (9 fields) — absorbs regulatory change without modifying core schema
- `governance_certification_level` in Liability Metadata — EMV-inspired proportional liability
- `interop_declarations` in Agent Cards — AP2/UCP/TAP compatibility signaling from a single discovery document
- Transport-agnostic `interaction_id` replacing A2A-specific `task_id` in correlation blocks

---

## Key Finding 1b: AFIX Passes the Lightweight Adoption Test

Post 1 in this series established that the prerequisite for any governance standard to achieve adoption is that it must be **extremely lightweight** — something any team with minimal knowledge can bolt on in weeks, not months. This was the lesson from FIX's rapid adoption (2 years from formation to widespread use) vs. RIXML's slow burn (16 years). AFIX is deliberately designed to pass this test:

| Adoption Pre-Requisite | How AFIX Meets It |
|---------------------------------------|----------------------|
| **Lightweight — no new infrastructure required** | Four JSON schemas. Not a new protocol, transport, or runtime. JSON Schema is the most widely understood schema format in software engineering. Any developer who can read an API spec can read AFIX. |
| **Bolt-on — works with what you already have** | **A2A:** Register five extensions in your existing Agent Card `extensions[]` array — a configuration change, not new protocol behavior. **MCP:** Add `_meta` fields to existing tool calls (convention, no protocol change) and add validation rules to your existing Kong/Apigee gateway — a configuration change to infrastructure you already operate. |
| **Minimal knowledge barrier** | AFIX uses only JSON Schema (draft 2020-12), standard HTTP headers, and existing protocol mechanisms. No proprietary formats, no new serialization, no custom tooling required. |
| **Incremental — adopt what you need, when you need it** | The Capability/Extension/Profile pattern means you do not implement all four components at once. Start with Agent Cards only (organizational identity — the minimum viable governance). Add Compliance Events when you need audit trails. Add Entitlement Interchange when sharing data across firms. Add Liability Metadata when operating multi-hop agent chains. Each is independently versioned. |
| **Proven pattern — not inventing new architecture** | UCP already ships the Capability/Extension/Profile pattern with multi-transport bindings. FHIR proved it in healthcare. FIX proved governance-in-message in trading. AFIX adapts established patterns, not novel ones. |

**The closing argument:** The Bloomberg/FactSet/LSEG triple-build proves the governance problem exists. The FIX/FHIR precedents prove it is solvable. AFIX's design proves it is solvable *simply* — four schemas, two bindings, existing infrastructure, incremental adoption. The question is not whether the standard is needed or whether it is implementable. The question is whether the industry coordinates before a hyperscaler ships a proprietary alternative.

---

## Key Finding 2: The Asymmetric Binding Is Precise and Documented

The binding specifications confirm the B.2.1 finding with engineering detail. The asymmetry is not a weakness — it is the core original contribution.

### A2A Binding: Protocol-Native ("FIX-like")

- **Five AFIX extensions** registered with versioned URIs (`https://finos.org/afix/v1/{component}`), four marked `required: true`
- **Extension negotiation** via `X-A2A-Extensions` header handshake — non-compliant clients rejected before task creation (error `-32001`)
- **Four attachment points:** Agent Card in A2A Agent Card `extensions[]`, Compliance Events in per-message `message.metadata`, Entitlement in per-task `task.metadata`, Liability/GHR in per-task `task.metadata`
- **Coexistence with AP2:** AFIX and AP2 mandates live in the same `extensions[]` arrays, each with its own namespaced URI
- **Four machine-verifiable task state invariants** — governance consistency rules enforced at the protocol level
- **Audit model:** Self-documenting. The message log IS the governance record.

### MCP Binding: Gateway-Mediated ("API Economy")

- **`_meta` field conventions** using `finos.org/afix/` vendor prefix on every `CallToolRequest`/`CallToolResult`
- **Gateway enforcement** via Kong ACLs, Apigee policies, or custom proxy rules — three configuration patterns provided
- **Agent Card discovery** via dedicated MCP tool (`afix_agent_card`) for immediate use + `.well-known` endpoint for future
- **SEP-2133 extension binding:** primary mechanism, with `_meta` as backward-compatible fallback
- **Audit model:** Infrastructure-documented. Requires correlating `_meta` traces + gateway logs + IdP logs.

### 11-Dimension Structural Comparison

The four most consequential asymmetries:

| Dimension | A2A | MCP | Implication |
|-----------|-----|-----|-------------|
| **Required enforcement** | Protocol-level (`required: true`) — structural | Gateway-level (policy config) — operational | A2A cannot be misconfigured away; MCP can |
| **Bilateral negotiation** | Both parties negotiate extensions | Server-side unilateral enforcement | A2A confirms mutual governance; MCP assumes it |
| **Per-message context** | Compliance Events on every message | Compliance Events on every request/response pair | Equivalent granularity, different cardinality |
| **Multi-hop propagation** | GHR appends to message metadata automatically | Orchestrator must manually copy `_meta` between tool calls | MCP has a silent failure mode A2A lacks |

**Four capabilities that favor MCP:** mandated transport auth (OAuth 2.1), per-tool scoping (governance varies by tool), gateway ecosystem (Kong/Apigee already deployed), tool annotations as risk vocabulary.

---

## Key Finding 3: The ACGP v2 Pilot Is Implementable

The pilot design progressed from an initial sketch to a seven-step, dual-transport architecture that tests the thesis directly.

### Pilot Architecture Summary

**Five participating entities:** Consumer Agent, Merchant MCP Server (UCP), Merchant A2A Agent, Payment Processor (AP2), Governance Observer (independent auditor).

**Seven steps (0-6):**

| Step | Activity | Transport | What This Tests |
|------|----------|-----------|-----------------|
| 0 | Discovery & Posture Exchange | MCP + A2A | Agent Card discovery on both transports; TAP identity verification |
| 1 | Product Discovery | MCP | Gateway-mediated governance; entitlement verification; redistribution control |
| 2 | Cart Negotiation | A2A | Protocol-native extension negotiation; `required: true` enforcement; AP2 coexistence |
| 3 | User Authorization | Client-side | Boundary between AFIX scope (cross-enterprise) and AP2 scope (user auth) |
| 4 | Payment Execution | A2A | AP2 + AFIX coexistence; `ap2_mandate_ref` traceability; GHR initiation |
| 5 | Settlement Confirmation | **MCP** | **Transport handoff (A2A → MCP mid-chain)**; GHR propagation across transports |
| 6 | Post-Purchase Dispute | A2A | Dispute adjudication from metadata alone; EMV liability shift; full audit reconstruction |

**The critical test is Step 5.** A governance chain starts on A2A (Step 4) and continues on MCP (Step 5). If the GHR hash chain survives the transport handoff with governance integrity intact, the protocol-agnostic claim is empirically validated. If it breaks, we learn exactly where the abstraction leaks.

### Success Criteria (Measurable)

Five categories of measurable criteria defined:
1. **Dual-transport:** Schema identity (SHA-256 hash match), semantic equivalence, cross-transport chain integrity
2. **Governance completeness:** All 5 components exercised on both transports
3. **Asymmetry validation:** Same schema → equivalent governance outcomes despite different enforcement
4. **Interoperability:** Dual-transport intermediary maintains governance continuity between A2A-only and MCP-only firms
5. **Performance:** <200ms governance overhead/step, <2s Agent Card verification, <50ms gateway enforcement, <30s entitlement revocation propagation, <5min full audit reconstruction

---

## Key Finding 4: The FIX/FHIR Analogy Is Article-Ready

Approximately 800 words of article-ready prose draw the FIX and FHIR parallels.

**FIX (1993) → A2A binding pattern.** Governance embedded in the message. ComplianceID, SolicitedFlag, and Parties block travel alongside execution data. The A2A binding follows the same principle — Compliance Events, Liability Metadata, and Entitlement references in `message.metadata` alongside AP2 mandates and UCP operations. The message IS the audit trail.

**FHIR (2011) → Capability/Extension/Profile pattern.** Capability-based architecture. Multi-transport binding (REST, Messaging, Documents, SMART on FHIR). Extension mechanism (domain-specific fields without modifying core). Profile constraints. AFIX mirrors this at every level.

**Where AFIX diverges:** Asymmetric binding. FIX had one transport (TCP). FHIR's transports were REST-adjacent. AFIX binds to two fundamentally different enforcement architectures. This has no precedent in either FIX or FHIR.

**Bloomberg/FactSet/LSEG as the pre-standardization parallel.** Before HL7v2, hospitals exchanged lab results through custom point-to-point interfaces. Bloomberg, FactSet, and LSEG are the financial services equivalent — three proprietary governance stacks on the same protocol. AFIX is the HL7 moment for agentic financial services.

---

## Key Finding 5: Production Decisions Are Resolved

### Schema Fragments: Inline vs. GitHub Repository

**Four inline fragments** (simplified, annotated, 15-20 lines each):
1. FS Agent Card — LEI, jurisdiction, authority thresholds, escalation chain
2. Compliance Event six-field core — the FIX/FHIR convergence point, with `ap2_mandate_ref`
3. A2A extensions coexistence — AFIX + AP2 in same `extensions[]`
4. MCP `_meta` governance — same schema, different attachment (side-by-side with #3)

**Six items for repository reference:** Full schemas (all 5 components), A2A binding walkthrough, MCP binding walkthrough (incl. Kong/Apigee config), AFIX Profile schema, full CEAG charter.

### AFIX Naming: Confirmed

"AFIX" stands. Five alternatives evaluated (FSGS, CEAG-S, OGS, AGS, FGML) and rejected. Positioning: "proposed governance schema specification" with a path to FINOS standardization — not a presumptive standard.

### FINOS Charter Excerpt

Article-ready excerpt drafted focusing on the "why standardize" argument — Bloomberg/FactSet/LSEG triple-build as evidence, design principle (format not policy), Year 1 deliverables, membership model. Full charter structure goes to appendix/repository.

---

## Key Finding 6: AP2 Cross-Reference Resolved

**Decision:** `ap2_mandate_ref` lives exclusively in the `ext/agentic-commerce` domain extension, accessed via `authorization._ext_refs` in Compliance Events. Removed from the base schema entirely — Phase A stress-testing confirmed the original design (optional field in base schema) violated protocol-agnosticism per IETF/W3C standards principles.

**Reasoning:** Follows UCP's AP2 Mandates Extension precedent. The field is a SHA-256 hash (opaque reference, not structural dependency). MCP-only deployments never encounter it. Creates auditable cross-protocol traceability in agentic commerce contexts without making the base AFIX schema dependent on AP2.

---

## Key Finding 7: Capability/Extension/Profile Pattern Designed

Following UCP's proven pattern:

- **Capabilities** independently versioned (e.g., Compliance Events v1.2, Entitlement Interchange v1.0)
- **Six domain extensions** defined: equity trading, fixed income, agentic commerce, wealth management, research distribution, payments
- **Profile declaration** at discoverable endpoint — server declares which capabilities and extensions it supports
- **Versioning policy:** additive-only within major versions, 18-month pre-announcement for breaking changes
- **Three illustrative profiles:** Bloomberg data provider, Jefferies trade recon, agentic commerce

---

## Open Questions and Risks

### Unresolved After This Workstream

1. **Who adjudicates proportional liability without a central authority?** Pre-negotiated (ISDA analog)? Neutral third party? Automated policy engine?
2. **How does the Governance Observer access MCP gateway logs?** Firms may not share internal infrastructure logs. Standard log export format needed?
3. **Does `chain_id` correlation work across organizations without a central registry?** SWIFT has a central tracker. Does AFIX need one?
4. **What happens when an MCP orchestrator drops `_meta` on multi-hop?** Silent governance chain break. Strongest weakness of MCP binding.
5. **How does entitlement revocation propagate to cached gateway policies?** 30-second SLA vs. real-world cache invalidation.
6. **Can the article present ACGP without reading as vaporware?** Recommended framing: "Here is the pilot design. Here is what it would prove. We are seeking consortium partners."

### Assumptions for Stress-Testing

1. Gateway enforcement ≈ protocol enforcement (functionally equivalent)
2. The asymmetry is presentable rather than disqualifying
3. `ap2_mandate_ref` doesn't violate protocol-agnosticism — **VERDICT: DOES NOT HOLD.** Moved to domain extension.
4. The six-field audit minimum is truly universal (verify against SOC 2, NIST)
5. EMV-inspired liability shift creates adoption incentive without enforcement authority — **VERDICT: PARTIAL.** ISDA replaces EMV as primary liability frame. EMV secondary for mechanism only.

### Strongest Counterarguments

1. "Standards bodies move too slowly for AI timescales" — RIXML took 16 years
2. "A2A and MCP may converge" — dual-binding becomes overengineered
3. "Governance latency kills adoption" — 1.2s added to a 6-step transaction
4. "Liability Metadata is legally unenforceable" — no court has adjudicated agent liability via metadata

### What Could Make This Wrong in 6 Months (top 3 by impact)

1. **Hyperscaler ships proprietary governance** (Medium prob / High impact) — Google already has AP2+UCP+A2A+MCP; adding governance is architecturally trivial
2. **Market doesn't need cross-enterprise governance yet** (Medium prob / High impact) — if agent transactions stay intra-firm, the problem doesn't materialize urgently
3. **Regulatory action defines different governance requirements** (Medium prob / Medium-High impact) — SEC/FCA requirements might not match AFIX design

---

## Original Contributions

### Technical
1. **Four AFIX schema components as concrete JSON Schemas** — publishable, implementable, not pseudocode
2. **A2A binding specification** — five extensions, four `required`, header negotiation, four task state invariants
3. **MCP binding specification** — SEP-2133 extension registration as primary, `_meta` conventions as backward-compatible fallback, three gateway enforcement patterns (Kong/Apigee/custom)
4. **11-dimension asymmetry analysis** — precise structural comparison, not hand-waving
5. **Capability/Extension/Profile pattern** — independently versioned capabilities, six domain extensions, profile declaration

### Analytical
6. **AP2 cross-reference as optional domain extension** — resolved design question with clear reasoning
7. **FIX/FHIR historical analogy as article-ready prose** — ~800 words positioning AFIX in established standardization patterns
8. **ACGP v2 seven-step dual-transport pilot** — implementable, with measurable success criteria across five categories
9. **Transport handoff test (Step 5)** — the single most important validation: GHR hash chain surviving A2A→MCP mid-chain

---

## Source Notes

| # | Research Note | Topic |
|---|------|-------|
| 1 | Binding specifications | AFIX schema components, A2A binding, MCP binding, asymmetry analysis, AP2 cross-reference, naming, capability/extension/profile pattern |
| 2 | Pilot and production decisions | ACGP v2 pilot architecture, success criteria, production decisions, FIX/FHIR analogy, open questions and risks |

**Combined with B.2.1:** 5 research notes + 2 synthesis documents across the full research workstream. The binding specs and pilot design draw on all three B.2.1 research notes (MCP, AP2/UCP, TAP) plus the earlier synthesis.
