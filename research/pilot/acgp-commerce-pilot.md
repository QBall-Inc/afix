# ACGP Use Case 2: Agentic Commerce Pilot
_Date: 2026-04-29 | Status: Design — Not Yet Executed_

---

## Purpose

This document specifies the second ACGP (Agentic Cross-Enterprise Governance Pilot) use case: a consumer purchase lifecycle that validates AFIX governance as a semantic layer across A2A and MCP protocols in an agentic commerce context. The pilot has not been executed. It is a design document specifying how participating entities would validate AFIX governance propagation across a multi-step commerce transaction, including a deliberate mid-chain transport handoff.

This use case complements [ACGP Use Case 1 (Research Workflow)](acgp-research-pilot.md), which validates AFIX in a financial services research context. Use Case 2 extends validation to commerce scenarios involving AP2 payment mandates, UCP checkout capabilities, and TAP identity verification — protocols that operate alongside AFIX in a consumer-facing transaction.

---

## 1. Pilot Architecture

### 1.1 Scenario

The pilot tests a consumer purchase lifecycle: a consumer's AI agent discovers a product, negotiates terms, authorizes payment, executes the transaction, and handles a post-purchase dispute. The scenario deliberately exercises all five AFIX governance components across both A2A and MCP transport bindings — not because a real transaction would necessarily use both, but because the pilot must validate that AFIX governance is transport-invariant.

### 1.2 Participating Entities

| Entity | Role | Transport(s) | AFIX Profile |
|--------|------|-------------|-----------------|
| **Consumer Agent** (e.g., Claude shopping assistant) | Shopping Agent — discovers products, negotiates, initiates payment on behalf of consumer | A2A (peer negotiation), MCP (tool calls) | Full stack: Agent Card, Compliance Events, Entitlement, Liability, GHR |
| **Merchant MCP Server** (e.g., Shopify + UCP) | Product catalog, checkout, order management | MCP (tool exposure via UCP MCP binding) | Agent Card, Compliance Events, Entitlement |
| **Merchant A2A Agent** (e.g., merchant negotiation agent) | Cart negotiation, pricing, terms | A2A (peer-to-peer) | Full stack |
| **Payment Processor** (e.g., Stripe/Adyen with AP2 integration) | Payment authorization and settlement | A2A (mandate exchange) | Compliance Events, Liability |
| **Governance Observer** (independent auditor — not grading own homework) | Real-time monitoring, post-hoc audit reconstruction | Read-only on both transports | Compliance Events (read), Liability (read), GHR (read) |

### 1.3 Full Step-by-Step Flow

**Step 0: Pre-Interaction Discovery and Governance Posture Exchange**

Before any transaction begins, agents discover each other and exchange governance postures.

- **Consumer Agent** fetches Merchant MCP Server's Agent Card via the `afix_agent_card` MCP tool.
- **Consumer Agent** fetches Merchant A2A Agent's Agent Card from `/.well-known/agent.json`, reading the `https://finos.org/afix/v1/agent-card` extension params.
- Both parties evaluate: regulatory jurisdiction compatibility (e.g., both under SEC/FINRA), compliance certification status (SOC2 current?), authority thresholds (max notional within range?), supported protocols (A2A and/or MCP?), `interop_declarations` (UCP capabilities, AP2 mandates support, TAP registration).
- **TAP verification (Visa agents):** If the Merchant's Agent Card declares `tap_registered: true`, the Consumer Agent initiates TAP identity verification — the three-signature handshake (Agent Intent, Consumer Recognition, Payment Information) per RFC 9421. This establishes cryptographic identity proof before commerce begins.
- **Governance Profile resolution:** Consumer Agent fetches the Merchant's AFIX Profile (capabilities, domain extensions, transport bindings) and computes the capability intersection per UCP's Discover-Negotiate-Fetch-Validate sequence.

**Compliance Events emitted:** `TASK_INITIATED` (Consumer Agent), `ENTITLEMENT_VERIFIED` (both parties verify mutual entitlements).

---

**Step 1: Product Discovery via MCP**

The Consumer Agent queries the Merchant's product catalog.

- **Transport:** MCP
- **Commerce Protocol:** UCP Checkout capability (MCP binding) — `tools/call` to merchant's `ucp_product_search` tool
- **AFIX Binding:** `_meta` field on `CallToolRequest` carries:
  - `finos.org/afix/agent-card` — Consumer Agent's LEI, agent_id, jurisdiction, Agent Card URI reference
  - `finos.org/afix/entitlement` — grant reference and requested scope (`DATA_ACCESS`, `PUBLIC` classification, `DISPLAY_ONLY` redistribution)
  - `finos.org/afix/compliance-event` — `DATA_ACCESSED` event with `LEVEL_4_SAFE` risk classification
- **Gateway enforcement (Merchant-side):** Kong/Apigee proxy validates `_meta` presence, verifies LEI against known-entity registry, checks entitlement grant validity, logs compliance event.
- **Response:** Merchant MCP Server returns product catalog as `structuredContent` with response `_meta` carrying server-side Compliance Event (outcome: SUCCESS, detail: "47 products returned") and entitlement confirmation (redistribution restriction: `DISPLAY_ONLY`).

**What this would test:** MCP binding — gateway-mediated governance enforcement. AFIX metadata in `_meta` field. Entitlement verification for data access. Redistribution control.

---

**Step 2: Cart Negotiation via A2A**

The Consumer Agent negotiates cart terms with the Merchant's A2A negotiation agent.

- **Transport:** A2A
- **Commerce Protocol:** UCP Checkout capability (A2A binding) — `SendMessage` creates an A2A task for cart assembly
- **AFIX Binding:** Protocol-native via A2A extensions:
  - Extension negotiation via `X-A2A-Extensions` header — Consumer Agent activates all four required AFIX extensions plus AP2 mandates extension.
  - Merchant A2A Agent validates: all `required: true` extensions present? If not, rejects with JSON-RPC error `-32001` before task creation.
  - Per-message Compliance Events in `message.metadata["https://finos.org/afix/v1/compliance-event"]` — `ACTION_EXECUTED` for cart operations (add item, apply coupon, calculate tax).
  - Task-level Entitlement reference in `task.metadata["https://finos.org/afix/v1/entitlement"]` — verified grant covering `ACTION_EXECUTION` scope for `CHECKOUT` category.
- **AP2 coexistence:** Merchant A2A Agent's `extensions[]` includes both AFIX compliance-event and AP2 mandates extension. At this step, AP2 is declared but not yet active — no payment mandate issued until Step 4.

**What this would test:** A2A binding — protocol-native governance with extension negotiation, `required` flag enforcement, per-message Compliance Events. Bilateral governance handshake. AFIX and AP2 coexistence in `extensions[]` arrays.

---

**Step 3: User Authorization (Client-Side)**

The consumer authorizes the purchase.

- **Transport:** Client-side (not cross-enterprise — runs on consumer's device)
- **Commerce Protocol:** AP2 — Consumer signs an Intent Mandate (human-not-present scenario) or Cart Mandate (human-present). Cryptographic signature via hardware-backed key on consumer device.
- **AFIX Activity:** None directly — this step runs on the consumer's device, outside cross-enterprise governance scope. However, the Consumer Agent emits a local Compliance Event (`PAYMENT_AUTHORIZED`, authorization status: `PENDING_HUMAN_REVIEW` → `AUTHORIZED`) that would be included in the next cross-enterprise message.
- **TAP interaction:** If TAP is active, the Consumer Recognition signature from Step 0 is bound to this authorization — the consumer identity proven by TAP matches the identity signing the AP2 mandate.

**What this would test:** The boundary between AFIX scope (cross-enterprise governance) and AP2 scope (user-level payment authorization). AFIX does not govern this step — it governs the organizational interaction that follows.

---

**Step 4: Payment Execution via A2A**

The Consumer Agent sends the signed AP2 mandate to the Payment Processor.

- **Transport:** A2A
- **Commerce Protocol:** AP2 Payment Mandate — derived from Step 3's Cart/Intent Mandate. The Payment Mandate signals agent involvement and operation mode to the payment network.
- **AFIX Binding:** Full governance envelope on the A2A message:
  - `message.metadata["https://finos.org/afix/v1/compliance-event"]` — `PAYMENT_AUTHORIZED` event. `authorization._ext_refs["https://finos.org/afix/ext/agentic-commerce/v1"].ap2_mandate_ref` carries SHA-256 hash of the AP2 Payment Mandate — the cross-protocol traceability link via domain extension. Risk classification: `LEVEL_2_HIGH_RISK`.
  - `task.metadata["https://finos.org/afix/v1/liability"]` — Liability Metadata with `liability_model: "CHAIN_OF_CUSTODY"`. Three parties declared: Consumer Agent org (ORIGINATOR), Merchant org (EXECUTOR), Payment Processor org (INTERMEDIARY). `governance_certification_level` declared per party — determines proportional liability under `PROPORTIONAL_BY_CERTIFICATION` fallback rule.
  - GHR (Governance Hop Record) — first hop: Consumer Agent → Payment Processor. Chain correlation ID (UUID, analogous to SWIFT UETR). Hop index: 1. Liability attestation, compliance event reference, compensation definition (type: FULL_REVERSAL, timeout: 30 days per Reg E).
- **AP2 on same message:** `message.metadata["https://ap2-protocol.org/v1/mandates"]` carries the Payment Mandate credential (SD-JWT-VC format). AP2 and AFIX coexist on the same message, each in their own namespaced metadata key.

**What this would test:** AP2 and AFIX coexistence on the same A2A message. Cross-protocol traceability via `ext/agentic-commerce` domain extension (`ap2_mandate_ref` in `_ext_refs`). Liability Metadata with multi-party allocation. GHR initiation. `LEVEL_2_HIGH_RISK` escalation (should trigger `AUTHORITY_ESCALATED` compliance event per task state invariants).

---

**Step 5: Payment Processor → Merchant Settlement via MCP**

The Payment Processor confirms settlement to the Merchant.

- **Transport:** MCP (demonstrating that a single transaction can span both transports)
- **Commerce Protocol:** UCP Order Management capability (MCP binding) — `tools/call` to merchant's `ucp_order_confirm` tool
- **AFIX Binding:** `_meta` on `CallToolRequest` carries:
  - `finos.org/afix/compliance-event` — `ACTION_EXECUTED` event (settlement confirmation). `correlation.chain_id` matches the GHR chain correlation ID from Step 4.
  - `finos.org/afix/liability` — Updated liability metadata reflecting settlement completion. All parties' `outcome.status: SUCCESS`.
  - `finos.org/afix/ghr` — Second hop record appended. Hop index: 2. Payment Processor → Merchant. Parent hash: SHA-256 of Step 4's GHR.
- **Gateway enforcement:** Merchant's Kong/Apigee validates `_meta` governance fields, cross-references chain_id against the A2A interaction from Step 4 (requires gateway access to the chain correlation registry).

**What this would test:** Transport handoff — a governance chain that started on A2A (Step 4) continues on MCP (Step 5). GHR propagation across transports. The same `chain_id` links A2A task metadata to MCP `_meta` fields. This is the asymmetry validation: does the governance record remain complete when the transport changes mid-chain?

---

**Step 6: Post-Purchase Dispute via A2A**

The consumer disputes the purchase. The Consumer Agent initiates a dispute with the Merchant A2A Agent.

- **Transport:** A2A
- **Commerce Protocol:** UCP Order Management capability (A2A binding) — dispute initiation
- **AFIX Binding:**
  - New A2A task created with fresh Compliance Event (`TASK_INITIATED`, `ANOMALY_DETECTED`). `correlation.chain_id` links back to the original transaction chain.
  - Liability Metadata activated with `liability_model: "PROPORTIONAL"` and `default_assignment.rule: "PROPORTIONAL_BY_CERTIFICATION"`. Under the ISDA-analog bilateral framework (Phase 1), the party with lower `governance_certification_level` absorbs proportionally more liability — the EMV proportional shift mechanism applied within ISDA-style bilateral terms.
  - GHR chain from Steps 4-5 attached as evidence. `dispute_resolution.evidence_chain` references ordered Compliance Event IDs forming the complete transaction audit trail.
  - AP2 mandate chain (Intent/Cart/Payment) referenced via `ext/agentic-commerce` domain extension's `ap2_mandate_ref` hashes (in `_ext_refs`) — proving consumer authorization at each step.
- **Governance Observer role:** The independent auditor can reconstruct the full governance state of the transaction from (a) A2A message logs (Steps 2, 4, 6) containing protocol-native AFIX metadata, and (b) MCP gateway logs (Steps 1, 5) containing `_meta` AFIX metadata plus gateway enforcement records. This is the audit reconstruction test.

**What this would test:** Dispute resolution using only governance metadata. ISDA-analog bilateral liability adjudication with EMV proportional shift mechanism. GHR as evidence chain across transports. Full audit reconstruction by independent observer. **Payments precedent:** Stripe's Compelling Evidence 3.0 implementation demonstrates that structured, machine-readable proof already shifts liability between counterparties in the card network — the governance metadata equivalent in payments.<sup>STRIPE-1</sup>

---

### 1.4 Pilot Step-to-Protocol Mapping Table

| Step | Activity | Transport | Commerce Protocol | AFIX Components Active | What This Would Test |
|------|----------|-----------|-------------------|--------------------------|---------------------|
| **0** | Discovery & Posture Exchange | MCP + A2A | — | Agent Card, Entitlement, Profile | Agent Card discovery on both transports; TAP identity verification |
| **1** | Product Discovery | MCP | UCP Checkout (MCP) | Agent Card, Compliance Events, Entitlement | MCP binding — gateway-mediated enforcement; redistribution control |
| **2** | Cart Negotiation | A2A | UCP Checkout (A2A) | Agent Card, Compliance Events, Entitlement | A2A binding — protocol-native extension negotiation; AFIX + AP2 coexistence |
| **3** | User Authorization | Client-side | AP2 Mandate | — (local only) | Boundary between AFIX scope and AP2 scope |
| **4** | Payment Execution | A2A | AP2 Payment Mandate | Compliance Events, Liability, GHR | AP2 + AFIX coexistence; multi-party liability; GHR initiation |
| **5** | Settlement Confirmation | MCP | UCP Order Management (MCP) | Compliance Events, Liability, GHR | Transport handoff (A2A → MCP mid-chain); asymmetry validation |
| **6** | Post-Purchase Dispute | A2A | UCP Order Management (A2A) | All five components | Dispute adjudication from metadata alone; EMV liability shift |

---

## 2. Success Criteria

### 2.1 Dual-Transport Validation Criteria

| Criterion | Measurement | Target |
|-----------|-------------|--------|
| **Governance schema identity** | AFIX schema hash identical on both transports | 100% match — zero divergence between A2A and MCP bindings |
| **Semantic equivalence** | Compliance Events on A2A and MCP for the same chain_id produce identical governance meaning | Observer reconstructs identical governance state from both transport traces |
| **Cross-transport chain integrity** | GHR chain maintains unbroken hash chain across A2A → MCP handoff | `parent_hash` correctly references previous hop; `chain_id` matches; hop indices increment without gaps |
| **A2A-specific enforcement** | Extension negotiation rejects non-compliant clients | 100% rejection with error code -32001 for missing required extensions |
| **MCP-specific enforcement** | Gateway rejects tool calls without required `_meta` governance fields | 100% rejection at gateway before reaching MCP server |

### 2.2 Governance Completeness Criteria

| AFIX Component | Steps Exercised | Completeness Test |
|-------------------|----------------|-------------------|
| **FS Agent Card** | 0, 1, 2 | Both parties' Agent Cards exchanged and validated on both transports |
| **Compliance Events** | 0, 1, 2, 4, 5, 6 | Every governance-relevant action has a corresponding Compliance Event with six-field minimum |
| **Entitlement Interchange** | 0, 1, 2 | Grant verified before first data access; redistribution restriction enforced; revocation tested mid-pilot |
| **Liability Metadata** | 4, 5, 6 | Three liability models exercised: CHAIN_OF_CUSTODY, ORIGINATOR_LIABLE, PROPORTIONAL_BY_CERTIFICATION |
| **GHR** | 4, 5, 6 | Multi-hop chain across two transports; hash chain verified; compensation definitions present |

### 2.3 Performance Criteria

| Metric | Target | Rationale |
|--------|--------|-----------|
| **Governance latency overhead per step** | <200ms | Advisory/settlement tier — execution-tier workflows are out of AFIX scope |
| **Agent Card verification** | <2s | Includes fetch, certificate chain validation, certification check |
| **Gateway enforcement latency (MCP)** | <50ms | Gateway policy evaluation must not become a bottleneck |
| **GHR chain validation** | <100ms per hop | Scales linearly with chain length |
| **Entitlement revocation propagation** | <30s | Emergency revocation to all enforcement points within SLA |

---

## 3. Open Questions and Risks

### 3.1 Unresolved

| Question | Why It Matters |
|----------|---------------|
| Who determines proportional liability split without a central authority? | Pre-negotiated in framework agreement (ISDA analog)? Computed by neutral third party? |
| How does the Governance Observer access MCP gateway logs? | Firms may not share gateway logs externally |
| Does `chain_id` correlation work across organizations? | Without a central tracker (SWIFT analog), each firm sees only its own hops |
| What happens when the orchestrator drops `_meta` on MCP multi-hop? | Governance chain breaks silently — strongest MCP binding weakness |

### 3.2 Strongest Counterarguments

1. **"Standards bodies move too slowly for AI timescales."** Real risk — but FIX v4.0 was adopted within 2 years because the pain was acute and participants were motivated. The regulatory calendar compresses the timeline.

2. **"A2A and MCP may converge, making dual-binding unnecessary."** Possible but not indicated — independent development continues. A protocol-agnostic schema simply has one binding instead of two.

3. **"No one will adopt governance that adds latency."** FIX adds latency to order routing. SWIFT gpi adds latency to payments. Both succeeded because governance benefit exceeded cost.

4. **"Liability Metadata is legally unenforceable."** The ISDA Master Agreement was legally untested when first published in 1987. The schema documents allocation; the contract makes it enforceable.

---

## Sources

- [Tier 1] A2A Protocol Specification v1.0.0, https://a2a-protocol.org/latest/specification/
- [Tier 1] A2A Extensions, https://a2a-protocol.org/latest/topics/extensions/
- [Tier 1] MCP Specification (2025-06-18), https://modelcontextprotocol.io/specification/2025-06-18/server/tools
- [Tier 1] STRIPE-1: Stripe, "Optimizing payments at scale," 2025, https://go.stripe.global/rs/072-MDK-283/images/Optimizing-payments-at-scale.pdf
- [Tier 1] FHIR R5 CapabilityStatement, https://hl7.org/fhir/R5/capabilitystatement.html
- [Tier 1] Visa TAP Specification, https://github.com/visa/trusted-agent-protocol
- [Tier 1] AP2 Specification, https://github.com/google-agentic-commerce/ap2
- [Tier 1] AFIX Schemas, https://github.com/QBall-Inc/afix/tree/main/schemas
