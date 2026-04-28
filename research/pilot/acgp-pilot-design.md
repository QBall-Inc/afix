# ACGP Pilot Design: Agentic Commerce Governance Pilot
_Date: 2026-04-20 | Status: Final_

---

## Purpose

This document delivers the definitive ACGP v2 pilot architecture: a seven-step, dual-transport test that validates AFIX governance as a semantic layer across A2A and MCP protocols. The pilot tests whether governance requirements are truly protocol-invariant — same schema, different enforcement — by constructing a realistic agentic commerce scenario that exercises all five AFIX components on both transports, including a deliberate mid-chain transport handoff.

---

## 1. Pilot Architecture

### 1.1 Scenario

The pilot tests a consumer purchase lifecycle: a consumer's AI agent discovers a product, negotiates terms, authorizes payment, executes the transaction, and handles a post-purchase dispute. The scenario deliberately exercises all four AFIX governance components across both A2A and MCP transport bindings — not because a real transaction would necessarily use both, but because the pilot must validate that AFIX governance is transport-invariant.

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

**What this tests:** MCP binding — gateway-mediated governance enforcement. AFIX metadata in `_meta` field. Entitlement verification for data access. Redistribution control.

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

**What this tests:** A2A binding — protocol-native governance with extension negotiation, `required` flag enforcement, per-message Compliance Events. Bilateral governance handshake. AFIX and AP2 coexistence in `extensions[]` arrays.

---

**Step 3: User Authorization (Client-Side)**

The consumer authorizes the purchase.

- **Transport:** Client-side (not cross-enterprise — runs on consumer's device)
- **Commerce Protocol:** AP2 — Consumer signs an Intent Mandate (human-not-present scenario) or Cart Mandate (human-present). Cryptographic signature via hardware-backed key on consumer device.<sup>AP2-6,7</sup>
- **AFIX Activity:** None directly — this step runs on the consumer's device, outside cross-enterprise governance scope. However, the Consumer Agent emits a local Compliance Event (`PAYMENT_AUTHORIZED`, authorization status: `PENDING_HUMAN_REVIEW` → `AUTHORIZED`) that will be included in the next cross-enterprise message.
- **TAP interaction:** If TAP is active, the Consumer Recognition signature from Step 0 is bound to this authorization — the consumer identity proven by TAP matches the identity signing the AP2 mandate.

**What this tests:** The boundary between AFIX scope (cross-enterprise governance) and AP2 scope (user-level payment authorization). AFIX does not govern this step — it governs the organizational interaction that follows.

---

**Step 4: Payment Execution via A2A**

The Consumer Agent sends the signed AP2 mandate to the Payment Processor.

- **Transport:** A2A
- **Commerce Protocol:** AP2 Payment Mandate — derived from Step 3's Cart/Intent Mandate. The Payment Mandate signals agent involvement and operation mode to the payment network.<sup>AP2-6,7</sup>
- **AFIX Binding:** Full governance envelope on the A2A message:
  - `message.metadata["https://finos.org/afix/v1/compliance-event"]` — `PAYMENT_AUTHORIZED` event. `authorization._ext_refs["https://finos.org/afix/ext/agentic-commerce/v1"].ap2_mandate_ref` carries SHA-256 hash of the AP2 Payment Mandate — the cross-protocol traceability link via domain extension. Risk classification: `LEVEL_2_HIGH_RISK`.
  - `task.metadata["https://finos.org/afix/v1/liability"]` — Liability Metadata with `liability_model: "CHAIN_OF_CUSTODY"`. Three parties declared: Consumer Agent org (ORIGINATOR), Merchant org (EXECUTOR), Payment Processor org (INTERMEDIARY). `governance_certification_level` declared per party — determines proportional liability under `PROPORTIONAL_BY_CERTIFICATION` fallback rule.
  - GHR (Governance Hop Record) — first hop: Consumer Agent → Payment Processor. Chain correlation ID (UUID, analogous to SWIFT UETR). Hop index: 1. Liability attestation, compliance event reference, compensation definition (type: FULL_REVERSAL, timeout: 30 days per Reg E).
- **AP2 on same message:** `message.metadata["https://ap2-protocol.org/v1/mandates"]` carries the Payment Mandate credential (SD-JWT-VC format). AP2 and AFIX coexist on the same message, each in their own namespaced metadata key.

**What this tests:** AP2 and AFIX coexistence on the same A2A message. Cross-protocol traceability via `ext/agentic-commerce` domain extension (`ap2_mandate_ref` in `_ext_refs`). Liability Metadata with multi-party allocation. GHR initiation. `LEVEL_2_HIGH_RISK` escalation (should trigger `AUTHORITY_ESCALATED` compliance event per task state invariants).

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

**What this tests:** Transport handoff — a governance chain that started on A2A (Step 4) continues on MCP (Step 5). GHR propagation across transports. The same `chain_id` links A2A task metadata to MCP `_meta` fields. This is the asymmetry validation: does the governance record remain complete when the transport changes mid-chain?

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

**What this tests:** Dispute resolution using only governance metadata. ISDA-analog bilateral liability adjudication with EMV proportional shift mechanism. GHR as evidence chain across transports. Full audit reconstruction by independent observer. The "100% dispute adjudication from metadata alone" criterion.

---

### 1.4 Pilot Step-to-Protocol Mapping Table

| Step | Activity | Transport | Commerce Protocol | AFIX Components Active | Governance Metadata Exchanged | What This Tests |
|------|----------|-----------|-------------------|--------------------------|-------------------------------|-----------------|
| **0** | Discovery & Posture Exchange | MCP (tool call) + A2A (`.well-known`) | — | Agent Card, Entitlement, Profile | FS Agent Cards (both parties), AFIX Profiles, Entitlement grants, TAP signatures (if applicable) | Agent Card discovery on both transports; governance posture evaluation; TAP identity verification; capability intersection resolution |
| **1** | Product Discovery | MCP | UCP Checkout (MCP binding) | Agent Card, Compliance Events, Entitlement | `_meta`: Agent Card reference, entitlement grant + requested scope (DATA_ACCESS, DISPLAY_ONLY), compliance event (DATA_ACCESSED, LEVEL_4_SAFE). Response `_meta`: server compliance event, redistribution restriction, metering data | MCP binding — gateway-mediated enforcement; `_meta` field governance; entitlement verification for data access; redistribution control |
| **2** | Cart Negotiation | A2A | UCP Checkout (A2A binding) | Agent Card, Compliance Events, Entitlement | `X-A2A-Extensions` header handshake; per-message compliance events in `message.metadata` (ACTION_EXECUTED per cart operation); task-level entitlement reference; AP2 extension declared but not active | A2A binding — protocol-native extension negotiation; `required: true` enforcement; per-message compliance events; AFIX + AP2 coexistence in `extensions[]` |
| **3** | User Authorization | Client-side | AP2 Intent/Cart Mandate | — (local only) | Consumer signs AP2 mandate; local compliance event (PAYMENT_AUTHORIZED, PENDING → AUTHORIZED); TAP Consumer Recognition binding | Boundary between AFIX scope (cross-enterprise) and AP2 scope (user authorization); governance layer separation |
| **4** | Payment Execution | A2A | AP2 Payment Mandate | Compliance Events, Liability, GHR | Compliance event (PAYMENT_AUTHORIZED, LEVEL_2_HIGH_RISK) with `ap2_mandate_ref` hash; Liability Metadata (CHAIN_OF_CUSTODY, 3 parties, certification levels); GHR hop 1 (chain_id, compensation: FULL_REVERSAL); AP2 Payment Mandate in parallel metadata key | AP2 + AFIX coexistence on same message; `ap2_mandate_ref` cross-protocol traceability; multi-party liability; GHR initiation; high-risk escalation invariant |
| **5** | Settlement Confirmation | MCP | UCP Order Management (MCP binding) | Compliance Events, Liability, GHR | `_meta`: compliance event (ACTION_EXECUTED, settlement); updated liability (SUCCESS); GHR hop 2 (parent_hash links to Step 4 GHR); `chain_id` correlation | Transport handoff (A2A → MCP mid-chain); GHR propagation across transports; `chain_id` linking A2A task metadata to MCP `_meta`; asymmetry validation |
| **6** | Post-Purchase Dispute | A2A | UCP Order Management (A2A binding) | All five components | New task: compliance events (TASK_INITIATED, ANOMALY_DETECTED); Liability (PROPORTIONAL, PROPORTIONAL_BY_CERTIFICATION); full GHR chain as evidence; AP2 mandate hashes as authorization proof; `dispute_resolution.evidence_chain` referencing all prior Compliance Event IDs | Dispute adjudication from metadata alone; EMV liability shift; GHR as evidence chain across transports; full audit reconstruction by Governance Observer |

---

## 2. Success Criteria

### 2.1 Dual-Transport Validation Criteria

| Criterion | Measurement | Target | How Verified |
|-----------|-------------|--------|--------------|
| **Governance schema identity** | AFIX schema hash (SHA-256 of each component's `$id`-resolved schema) is identical on both transports | 100% match — zero schema divergence between A2A and MCP bindings | Automated: schema validator compares governance metadata extracted from A2A `message.metadata` vs. MCP `_meta` for the same component type |
| **Semantic equivalence** | A Compliance Event emitted on A2A (Step 4) and a Compliance Event emitted on MCP (Step 5) for the same chain_id produce identical governance meaning when evaluated by the Governance Observer | Observer reconstructs identical governance state from both transport traces | Manual: Governance Observer writes audit report from A2A-only trace, then from MCP-only trace, then from combined. Reports must reach same conclusions. |
| **Cross-transport chain integrity** | GHR chain that starts on A2A (Step 4) and continues on MCP (Step 5) maintains unbroken hash chain and correct hop indexing | `parent_hash` on MCP hop correctly references SHA-256 of A2A hop; `chain_id` matches; hop indices increment without gaps | Automated: chain validator walks GHR records across both transport logs and verifies hash chain integrity |
| **A2A-specific enforcement** | Extension negotiation (`X-A2A-Extensions` header) correctly rejects non-compliant clients and accepts compliant ones | 100% of interactions without required AFIX extensions rejected with error code -32001 before task creation | Test case: send `SendMessage` without AFIX extensions to a `required: true` agent. Expect rejection. |
| **MCP-specific enforcement** | Gateway (Kong/Apigee) correctly rejects tool calls without required `_meta` governance fields | 100% of tool calls without `_meta` governance fields rejected at gateway before reaching MCP server | Test case: send `tools/call` without `finos.org/afix/agent-card` in `_meta`. Expect 403 from gateway. |

### 2.2 Governance Completeness Criteria

| AFIX Component | Steps Exercised | Completeness Test |
|-------------------|----------------|-------------------|
| **FS Agent Card** | 0, 1, 2 | Both parties' Agent Cards exchanged and validated on both transports. Regulatory jurisdiction evaluated. Compliance certifications checked for expiry. Authority thresholds enforced (no transaction above declared max_notional_usd). |
| **Compliance Events** | 0, 1, 2, 4, 5, 6 | Every state transition and governance-relevant action has a corresponding Compliance Event. Six-field minimum (actor, action, target, timestamp, authorization, outcome) present on every event. Event types exercised: TASK_INITIATED, ENTITLEMENT_VERIFIED, DATA_ACCESSED, ACTION_EXECUTED, PAYMENT_AUTHORIZED, ANOMALY_DETECTED, LIABILITY_ASSIGNED. |
| **Entitlement Interchange** | 0, 1, 2 | Entitlement grant verified before first data access (Step 1) and first action execution (Step 2). Redistribution restriction enforced (DISPLAY_ONLY on product catalog data). Revocation tested: mid-pilot, revoke Consumer Agent's entitlement and verify enforcement within revocation SLA (default 30 seconds). |
| **Liability Metadata** | 4, 5, 6 | Three liability models exercised: CHAIN_OF_CUSTODY (Step 4, payment), ORIGINATOR_LIABLE (default fallback), PROPORTIONAL_BY_CERTIFICATION (Step 6, dispute). All parties have declared governance_certification_level. Liability closure invariant: every COMPLETED task has Liability Metadata. |
| **GHR (Governance Hop Record)** | 4, 5, 6 | Multi-hop chain created across two transports. Hash chain verified. Chain correlation ID (UETR-analog) propagated. Compensation definitions present at each hop. Evidence chain references in dispute resolution (Step 6) correctly point to all prior GHR records. |

**Completeness target:** All five AFIX components exercised at least once on each transport (A2A and MCP). All four task state invariants validated.

### 2.3 Asymmetry Validation Criteria

The pilot's most intellectually significant test: does the same AFIX schema produce *equivalent* governance outcomes despite structurally different enforcement on each transport?

| Asymmetry Dimension | A2A Behavior | MCP Behavior | Equivalence Test |
|---------------------|-------------|-------------|-----------------|
| **Extension enforcement** | Protocol-level rejection (`required: true`) | Gateway-level rejection (Kong/Apigee policy) | Both must reject non-compliant clients. Rejection latency must be <100ms for both. The *mechanism* differs; the *outcome* (non-compliant client cannot interact) must be identical. |
| **Governance negotiation** | Bilateral (`X-A2A-Extensions` handshake) | Unilateral (server-side gateway enforcement) | Both must establish a governance contract before interaction proceeds. |
| **Audit trail location** | Self-documenting — governance metadata IN the message log | Infrastructure-documented — `_meta` + gateway logs + IdP logs | Governance Observer must be able to reconstruct complete governance state from either approach. Both reconstructions must yield the same governance conclusions. |
| **Per-interaction granularity** | Per-Message and per-Artifact metadata (4 attachment points) | Per-request and per-response `_meta` (2 attachment points) | For multi-step A2A tasks (Step 2: multiple cart operations), each message carries its own Compliance Event. For equivalent MCP operations, each tool call is a separate request/response pair. |
| **Multi-hop propagation** | GHR appends to message metadata; chain travels with message | Orchestrator must copy `_meta` GHR between tool calls | Hash chain integrity identical? Hop count identical? Any governance data lost in MCP propagation that A2A preserved automatically? |

### 2.4 Interoperability Criteria

**Test scenario:** Firm A implements only the A2A binding. Firm B implements only the MCP binding. Can they interact with governance intact?

The interoperability question is: can a **dual-transport intermediary** (the Consumer Agent, which supports both A2A and MCP) maintain governance continuity when interacting with A2A-only and MCP-only counterparties in the same transaction chain?

| Criterion | Measurement | Target |
|-----------|-------------|--------|
| **Chain correlation** | `chain_id` links governance records from A2A-only firm (Step 4) and MCP-only firm (Step 5) into one audit trail | Single `chain_id` spans both transports; Governance Observer can query by `chain_id` and retrieve all records regardless of originating transport |
| **Liability chain completeness** | Liability Metadata correctly references parties across both transports | All parties appear in the Liability Metadata `parties` array with correct roles and certification levels |
| **Schema compatibility** | A firm that built AFIX for A2A only can read and validate a Compliance Event that originated on MCP (and vice versa) | The JSON schema validation passes: a Compliance Event extracted from MCP `_meta` validates against the same `$id: https://finos.org/afix/v1/compliance-event` schema as one extracted from A2A `message.metadata` |

### 2.5 Performance Criteria

| Metric | Target | Rationale |
|--------|--------|-----------|
| **Governance latency overhead per step** | <200ms | Governance metadata creation, attachment, and validation should not measurably degrade transaction UX. 200ms is perceptible but not blocking. Applies to advisory, research, compliance, and settlement tiers. Execution-tier workflows (matching engines, order routing at microsecond latency) are out of scope for AFIX. |
| **Agent Card verification** | <2s | Includes Agent Card fetch, certificate chain validation, compliance certification check, authority threshold evaluation. |
| **Gateway enforcement latency (MCP only)** | <50ms | Gateway policy evaluation (validate `_meta` presence, verify LEI, check entitlement) must not become a bottleneck. Kong AI Gateway benchmarks sub-10ms for ACL checks. |
| **GHR chain validation** | <100ms per hop | Hash chain verification, hop index validation, correlation ID lookup. Scales linearly with chain length. |
| **Entitlement revocation propagation** | <30s | Emergency revocation must propagate to all enforcement points (A2A agents, MCP gateways) within SLA. |
| **Audit reconstruction time (Governance Observer)** | <5min for full transaction | Governance Observer can reconstruct complete governance state of a 6-step, dual-transport transaction from stored logs/metadata within 5 minutes. Measures operational audit feasibility. |

---

## 3. FIX/FHIR Historical Analogy

_This section is article-ready prose._

### Standards That Standardized the Envelope, Not the Decision

AFIX follows a pattern that has played out twice before in industries that needed cross-enterprise coordination under regulatory pressure.

**FIX Protocol (1993): Governance Embedded in the Message**

When the Financial Information eXchange protocol standardized broker-dealer order routing in the early 1990s, it did not tell firms how to trade. It told them how to *describe* trades in a format every counterparty could parse. An ExecutionReport (MsgType=8) carries ComplianceID, SolicitedFlag, and a structured Parties block alongside the execution data — governance metadata embedded in the same message as the trade, not logged in a separate system.<sup>B1-3</sup>

This is the A2A binding pattern. In A2A with AFIX extensions, Compliance Events, Liability Metadata, and Entitlement references travel in the same `message.metadata` as the business content — AP2 payment mandates, UCP checkout operations. The message IS the audit trail. A compliance officer can reconstruct the governance state of any interaction by reading the A2A message log, just as a post-trade compliance system reconstructs order flow from FIX drop copies.

FIX's enforcement was also embedded: a SenderCompID/TargetCompID mismatch prevents session establishment; a missing required field is a protocol error, not a policy suggestion. A2A's `required: true` extension flag follows the same principle — non-compliant clients are structurally prevented from interacting before any business logic executes.<sup>A2A-1</sup>

**HL7 FHIR (2011): Capability-Based Design, Multiple Transports, Extension Mechanism**

When HL7 developed FHIR to replace the aging HL7v2 standard, it faced a problem structurally identical to AFIX's: hundreds of institutions exchanging sensitive data through incompatible proprietary integrations, with regulatory requirements (HIPAA, Meaningful Use) demanding interoperability.

FHIR's answer was a capability-based architecture.<sup>FHIR-1</sup> Each server declares a CapabilityStatement — which resources it supports, which operations, which profiles. Clients discover capabilities before interaction. AFIX's Profile declaration (capabilities, domain extensions, transport bindings, all independently versioned) follows FHIR's exact pattern, adapted through UCP's proven commercial implementation.

FHIR also pioneered the multi-transport binding that AFIX adopts. The same clinical resource (Patient, Observation, MedicationRequest) can be exchanged via RESTful HTTP, FHIR Messaging (asynchronous), FHIR Documents (bundled), or SMART on FHIR (app-embedded). The resource schema is transport-invariant; the binding specification defines how it attaches to each transport. AFIX governance components are transport-invariant; the A2A binding and MCP binding define how they attach.

FHIR's extension mechanism is perhaps the most directly relevant precedent. Core FHIR resources define a minimum viable schema. Extensions add domain-specific fields without modifying core. Profiles constrain allowed values for specific use cases. AFIX mirrors this at every level: core governance components (Agent Card, Compliance Event, Entitlement, Liability), domain extensions (`ext/equity-trading`, `ext/agentic-commerce`), and profiles that constrain values for specific regulatory jurisdictions.

**Where AFIX Diverges: Asymmetric Binding**

FIX had one transport — the FIX session protocol over TCP. Every participant ran the same transport with the same enforcement model. FHIR's transports were REST-adjacent — HTTP variants with similar enforcement characteristics.

AFIX binds to two transports with fundamentally different enforcement architectures. A2A provides protocol-native governance: typed extensions, bilateral negotiation, `required` flag enforcement, and self-documenting audit trails in message metadata.<sup>A2A-1,2</sup> MCP provides gateway-mediated governance: untyped `_meta` metadata by convention (with SEP-2133 extension negotiation as the connection-level enforcement), and audit trails distributed across protocol traces and gateway logs.<sup>MCP-1,8</sup>

This asymmetry has no precedent in FIX or FHIR. It is a structural consequence of binding to two transports designed for different interaction patterns — A2A for peer negotiation, MCP for client-server tool invocation. The governance *meaning* is identical on both transports. The governance *evidence* is located differently. For regulators, A2A's self-documenting model is easier to audit. For operators, MCP's gateway-mediated model is easier to deploy — they already have the infrastructure.

**The Pre-Standardization Parallel: Bloomberg, FactSet, LSEG**

Before HL7v2 standardized clinical data exchange, hospitals transmitted lab results through custom point-to-point interfaces — each pair of institutions negotiating its own message format, transport, and error handling.

Bloomberg, FactSet, and LSEG are the financial services equivalent. Each has independently built proprietary governance middleware on top of MCP — identity management, access control, audit logging, rate limiting, entitlement enforcement — because the protocol does not carry this metadata natively.<sup>MCP-4,5,7</sup> Three firms, same governance categories, three proprietary implementations. Every new firm deploying MCP for agent interactions will build the same middleware a fourth time, a fifth time, a sixth time.

AFIX is the HL7 moment for agentic financial services: the transition from proprietary point-to-point governance to a shared schema that lets firms focus on their governance *policies* rather than their governance *plumbing*. FIX proved the pattern in trading. FHIR proved it in healthcare. The question is not whether financial services needs this standard — three independent proprietary implementations have answered that. The question is how long the industry waits before building it.

---

## 4. Open Questions and Risks

### 4.1 Unresolved

| Question | Why It Matters | Where It Gets Resolved |
|----------|---------------|----------------------|
| **Who determines proportional liability split without a central authority?** | The `PROPORTIONAL_BY_CERTIFICATION` rule requires someone to evaluate certification levels and compute proportions. Pre-negotiated in framework agreement (ISDA analog)? Computed by neutral third party? Automated by policy engine? | Requires the CEAG working group to define adjudication rules as part of the Liability Metadata specification. |
| **How does the Governance Observer access MCP gateway logs?** | The pilot assumes the Observer can correlate MCP `_meta` traces with gateway enforcement logs. In practice, gateway logs are internal infrastructure — firms may not share them. | Pilot Phase 1 (Schema Validation). Requires defining a standard governance log export format, or mandating that gateway enforcement decisions are reflected back in response `_meta`. |
| **Does `chain_id` correlation work across organizations?** | The GHR uses a UUID `chain_id` analogous to SWIFT UETR. But SWIFT has a central tracker service. Who maintains the chain correlation registry for AFIX? | Pilot Phase 2 (Lifecycle Integration). Start with firm-local correlation registries; evaluate shared registry need based on pilot results. |
| **What happens when the orchestrator drops `_meta` on MCP multi-hop?** | If the Consumer Agent (orchestrator) fails to propagate `_meta` governance fields from one MCP tool call's response to the next tool call's request, the governance chain breaks silently. | Strongest weakness of MCP binding. May require a "governance-aware orchestrator" certification as part of AFIX compliance. |
| **How does entitlement revocation propagate to MCP gateways?** | The spec says 30-second default SLA. But gateways cache policies. What is the actual propagation path from revocation event to gateway policy update? | Pilot Phase 1. Requires testing with actual Kong/Apigee deployments to measure real-world propagation latency. |

### 4.2 Assumptions Needing Verification

| Assumption | Claim | Strongest Counter | Test Design |
|-----------|-------|-------------------|-------------|
| **Gateway enforcement is functionally equivalent to protocol enforcement** | "Same outcome, different enforcement locus" — the article claims A2A and MCP produce equivalent governance despite different enforcement. | Gateway misconfiguration is a real and common failure mode. Protocol enforcement (A2A's `required: true`) is structural — it cannot be misconfigured away. | Test: deliberately misconfigure a Kong gateway. Does governance fail silently? Compare: deliberately misconfigure an A2A agent. Does governance fail loudly? |
| **The asymmetry is presentable rather than disqualifying** | The A2A/MCP asymmetry is framed as a finding, not a problem. But a critic could argue: "If MCP governance is weaker, why not just mandate A2A for cross-enterprise interactions?" | MCP has the deployed infrastructure (Bloomberg, FactSet, LSEG, 8+ bank MCP servers). A2A has the governance architecture but minimal finserv deployment. | Present the steelmanned argument for A2A-only governance and the steelmanned argument for MCP-only governance, then show why dual-binding is the pragmatic answer. |
| **The six-field audit minimum is truly universal** | FIX, FHIR, and RIXML converge on six fields — presented as empirical evidence. | Three data points is pattern-spotting, not proof. Other audit standards may define different minimal field sets. | Verify against SOC 2 and NIST audit control frameworks. If they define additional mandatory fields not in the six-field minimum, document the gap. |

### 4.3 Strongest Counterarguments

**1. "Standards bodies move too slowly for AI timescales."**
RIXML took 16 years. HL7 FHIR took 7 years from R1 to widespread adoption. AI agent deployments are scaling in months.

_Counter:_ FIX v4.0 was adopted within 2 years of formation because the pain was acute and the participants were motivated. The regulatory calendar (EU AI Act enforcement August 2026, anticipated SEC guidance) compresses the timeline. But the risk is real — execution speed is existential for standards in fast-moving markets.

**2. "A2A and MCP may converge, making dual-binding unnecessary."**

_Counter:_ A2A and MCP serve different interaction patterns. Convergence is possible but not indicated — independent development continues on both protocols. Even if they converge, a protocol-agnostic governance schema simply has one binding instead of two. The risk is wasted engineering effort, not architectural failure.

**3. "No one will adopt governance that adds latency to agent transactions."**
200ms overhead per step means 1.2 seconds added to a 6-step transaction.

_Counter:_ FIX adds latency to order routing. SWIFT gpi adds latency to cross-border payments. Both succeeded because the governance benefit exceeded the cost. The <200ms target must be achieved — if governance overhead is materially higher, adoption will stall. The target explicitly excludes execution-tier workflows (matching engines, microsecond order routing), which remain governed by existing protocol-specific mechanisms.

**4. "Liability Metadata is legally unenforceable."**
No court has adjudicated agent liability using metadata-defined allocation.

_Counter:_ The ISDA Master Agreement was legally untested when first published in 1987. It became the contractual backbone of the derivatives market because courts accepted well-documented bilateral agreements. AFIX Liability Metadata + the Agent Interaction Agreement (contractual wrapper) follows the same developmental path. The schema documents the allocation; the contract makes it enforceable. This is a known-open question, not a design flaw.

### 4.4 What Could Make This Wrong in 6 Months

| Risk | Probability | Impact | What Would Happen |
|------|-------------|--------|-------------------|
| **A major cloud provider ships a proprietary governance standard** | MEDIUM | HIGH | If Google, Microsoft, or AWS ships an "Agent Governance API" with hyperscaler distribution, AFIX becomes a competitor rather than the default. Mitigation: AFIX's open-source, FINOS-hosted positioning is its defense — finserv firms are unlikely to adopt a single-vendor governance standard for cross-enterprise use. |
| **MCP SEP-2133 extension ecosystem matures to include governance primitives** | LOW | MEDIUM | If MCP's extension ecosystem develops governance-specific extensions natively, AFIX on MCP could become redundant. Mitigation: SEP-2133 is a framework for extensions, not specific extensions. AFIX registers as an extension under SEP-2133. The framework's existence validates the approach; it does not compete with it. |
| **A regulatory enforcement action defines agent governance requirements** | MEDIUM | MEDIUM-HIGH | If SEC or FCA issues specific agent governance requirements before CEAG forms, the requirements may not match AFIX's design. Mitigation: AFIX is designed to be policy-agnostic (format, not values). Regulatory requirements would constrain *what values go in the schema*, not *the schema structure*. But if regulators specify a different metadata format entirely, AFIX must adapt or become irrelevant. |
| **The market does not actually need cross-enterprise agent governance yet** | MEDIUM | HIGH | If agent transactions remain primarily intra-firm for the next 12 months, the cross-enterprise governance problem does not materialize urgently enough to drive standardization. Mitigation: the forcing function is real — 8 banks with MCP servers, 3 payment networks with agent commerce frameworks, all launched in Q1-Q2 2026. Cross-enterprise interaction is not hypothetical. But the *volume* may remain low enough that proprietary point-to-point solutions suffice for another year. |

---

## 5. Analyst Assessment

### On the Pilot

The ACGP v2 design is significantly stronger than the initial version. The dual-transport architecture makes the pilot test the thesis — "governance is a semantic layer, not a transport layer" — rather than merely demonstrating that governance metadata can attach to messages. The most valuable test is Step 5 (settlement confirmation via MCP mid-chain): if the GHR hash chain survives a transport handoff from A2A to MCP with governance integrity intact, the protocol-agnostic claim is empirically validated. If it breaks, we learn exactly where the abstraction leaks.

The weakest point is Step 6 (dispute resolution). The success criterion — "100% dispute adjudication from metadata alone" — is aspirational in a pilot that has never been run. In a real dispute, metadata completeness depends on every participant correctly implementing governance metadata emission. One non-compliant participant breaks the chain. The pilot's controlled environment masks this problem. Flag it honestly in the article: this criterion tests the ideal case, not the degraded case.

### On Risks

The risk that weighs most heavily: **a hyperscaler ships proprietary governance first.** Google already has AP2, UCP, A2A, and MCP. Adding a governance layer to that stack would be architecturally trivial and distributionally devastating — every Google Cloud customer gets it for free. The defense is FINOS hosting and multi-vendor governance, but that defense only works if CEAG forms before the proprietary alternative ships. Speed matters more than completeness for v1.0.

The risk that weighs least: **MCP and A2A converge.** The protocols serve genuinely different interaction patterns. Convergence would require one pattern to subsume the other — possible but not likely on a 6-month horizon.

**Confidence in overall approach: HIGH.** The binding specifications are concrete, the pilot is implementable, the historical precedents are genuine, and the market evidence (Bloomberg/FactSet/LSEG triple-build) is the strongest argument. What the article needs most is honest acknowledgment of what is unproven: liability enforceability, proportional split adjudication, real-world multi-hop governance chain integrity. AFIX should be presented as a well-designed proposal backed by market evidence, not as a finished standard.

---

## Sources

### A2A Protocol
- [Tier 1] A2A-1: A2A Protocol, "Extensions," https://a2a-protocol.org/latest/topics/extensions/
- [Tier 1] A2A-2: A2A Protocol, "Specification v1.0.0," https://a2a-protocol.org/latest/specification/

### MCP Protocol
- [Tier 1] MCP-1: MCP Specification (2025-06-18), "Tools — Server," https://modelcontextprotocol.io/specification/2025-06-18/server/tools
- [Tier 1] MCP-4: Bloomberg LP, "Closing the Agentic AI productionization gap," https://www.bloomberg.com/company/stories/closing-the-agentic-ai-productionization-gap-bloomberg-embraces-mcp/
- [Tier 1] MCP-5: Skywork AI, "A Deep Dive into Bloomberg and MCP," https://skywork.ai/skypage/en/A-Deep-Dive-into-Bloomberg-and-MCP-The-blpapi-mcp-Server-for-Financial-AI-Agents/1972482843412180992
- [Tier 1] MCP-7: LSEG, "Scaling AI in Financial Services with LSEG's Trusted AI Ready Content and MCP," https://www.lseg.com/en/insights/scaling-ai-financial-services-with-lseg-trusted-ai-ready-content-mcp
- [Tier 1] MCP-8: Kong Inc., "Kong Introduces MCP Registry in Kong Konnect," https://www.prnewswire.com/news-releases/kong-introduces-mcp-registry-in-kong-konnect-302676451.html

### AP2/UCP
- [Tier 1] AP2-6: AP2 Specification, "Mandates — Intent, Cart, Payment," https://github.com/google-agentic-commerce/ap2
- [Tier 1] AP2-7: AP2 Specification, "Security Model — Cryptographic Signing," https://github.com/google-agentic-commerce/ap2

### TAP
- [Tier 1] TAP-1: Visa Developer, "Trusted Agent Protocol Specification," https://github.com/visa/trusted-agent-protocol
- [Tier 1] TAP-2: Visa, "TAP — Three-Signature Model," RFC 9421 HTTP Message Signatures

### FHIR
- [Tier 1] FHIR-1: HL7 FHIR R5, "CapabilityStatement Resource," https://hl7.org/fhir/R5/capabilitystatement.html
