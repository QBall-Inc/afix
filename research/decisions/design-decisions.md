# Design Decisions Log
_Date: 2026-04-25 | Status: Final_

This document captures confirmed design decisions made during the interactive brainstorm session that produced the Post 2 drafting blueprint. All decisions are listed with rationale, sources, and alternatives considered.

---

## Schema Design Decisions

### Decision 1: `ap2_mandate_ref` — Move to Domain Extension

**Decision:** `ap2_mandate_ref` lives exclusively in the `ext/agentic-commerce` domain extension, accessed via `authorization._ext_refs` in Compliance Events. Removed from the base schema entirely.

**Context:** Phase A stress-testing confirmed the original design (optional field in base schema) violated protocol agnosticism per IETF/W3C standards principles. Even as an optional opaque hash, an AP2-specific field in the base schema creates conceptual coupling. MCP-only deployments never encounter AP2; they should not encounter AP2 fields in the base schema.

**Reasoning:** Follows UCP's AP2 Mandates Extension precedent. The field is a SHA-256 hash (opaque reference, not structural dependency). MCP-only deployments never encounter it. Creates auditable cross-protocol traceability in agentic commerce contexts without making the base AFIX schema dependent on AP2.

**Article treatment:** Section 4 presents base schema without `ap2_mandate_ref`. Brief honest callout: "We caught this during stress-testing and refactored." Strengthens credibility.

**Alternatives considered:** Keep as optional field with a clear note that it is AP2-specific. Rejected: even optional fields with protocol-specific semantics create conceptual coupling that violates the protocol-agnosticism design principle.

---

### Decision 2: Six-Field Audit Core — Two-Tier Model

**Decision:** "Six fields = interoperability schema, nine fields + constraints = regulatory compliance schema." Core in article body; compliance extension referenced in the regulatory story.

**Context:** Phase A stress-test (Assumption 4) confirmed that the six-field minimum survives as the interoperability floor but does NOT meet regulatory compliance requirements standalone.

**Missing for compliance:**
- `model_id` + `dataset_version` — EU AI Act Article 12 logging requirements
- Microsecond timestamps — MiFID II RTS 25 (clock synchronization requirements for high-frequency trading venues)
- `log_integrity_hash` — EU AI Act Article 12 + SEC Rule 17a-4 tamper-evidence requirements

**Reasoning:** The two-tier model is architecturally correct: a stable six-field interoperability core that any firm can implement, plus a regulatory compliance extension that absorbs regulatory change without modifying the core. This also answers the regulatory change velocity problem (Finding 5 from Phase C review): regulations change faster than standards bodies; domain extensions are the escape valve.

**Alternatives considered:** Single schema with all nine fields required. Rejected: imposes regulatory burden on firms that do not need it (e.g., non-EU, non-MiFID deployments). Makes the schema jurisdiction-specific rather than universal.

---

### Decision 3: Liability Frame — ISDA Primary, EMV Secondary

**Decision:** ISDA as primary analogy for Phase 1 (bilateral master agreements). Basel peer-monitoring for Phase 2. EMV becomes secondary reference for the proportional shift *mechanism* only.

**Context:** Phase A stress-test (Assumption 5) showed that EMV liability shift worked because card networks had enforcement authority. No equivalent authority exists in a voluntary standards body. ISDA's bilateral master agreement model is the right framing for Phase 1 because the target audience lives in the ISDA world and will recognize the pattern immediately.

**Phased model:**
- **Phase 1 (years 1-2):** ISDA-analog bilateral agreements with AFIX Liability Metadata as the structured format. Works because early adopters are large banks with existing bilateral relationships.
- **Phase 2 (years 2-4):** Consortium-operated adjudication for pilot participants. Small enough for trust, large enough to validate.
- **Phase 3 (years 4+):** Automated policy engine for routine cases below a defined threshold, human escalation for complex disputes.

**Reasoning:** ISDA framing is more honest about the Phase 1 mechanism (bilateral, contractual) and more credible to the target audience. EMV's proportional shift *mechanism* (higher certification = lower liability proportion) is still valuable as a design precedent — it just should not be the primary framing because it implies enforcement authority that does not exist.

**Alternatives considered:** Rely entirely on regulatory mandate for liability enforcement. Rejected: regulatory mandate may not materialize in the timeframe, and the article should not depend on it.

---

### Decision 4: GHR Correlation Key — LEI:chain_id:hop_index

**Decision:** Correlation key is `LEI:chain_id:hop_index`, not `chain_id` alone. No central chain registry required.

**Design:**
- `chain_id` — originator-generated UUID v4, stays constant across the full chain. Identifies *which* governance chain.
- LEI — from the acting firm's FS Agent Card at each hop. Identifies *who* is acting. Already centrally issued by GLEIF — piggybacking on existing infrastructure.
- `hop_index` — sequences the hops within the chain.

**Multi-hop liability attribution:** Each GHR hop entry includes the acting institution's LEI as a required field. The full chain reads as a sequence of institutionally attributed governance events:

```
Hop 0: LEI_FirmA : chain_id : 0  (origination)
Hop 1: LEI_FirmB : chain_id : 1  (delegation)
Hop 2: LEI_FirmC : chain_id : 2  (execution)
```

Liability is attributable at the hop level, not just the chain level. Proportional liability flows toward the weaker governance posture at the relevant hop — same logic as EMV liability shift in card payments.

**Aggregator indexing:** Two axes — by `chain_id` (give me the full chain) and by LEI (give me all hops involving Firm X). Liability reconstruction walks the chain hop by hop.

**Schema impact:** The GHR design in binding specs has `chain_id`, `hop_index`, and `parent_hash` but did NOT originally include the acting institution's LEI as a per-hop field. This was added as a required field in each GHR hop entry.

**Alternatives considered:** Central chain registry (SWIFT UETR model). Rejected: requires infrastructure that does not exist and introduces a central point of control. The LEI:chain_id:hop_index composite key achieves correlation without a registry by piggybacking on GLEIF's existing infrastructure.

---

### Decision 5: MCP Extension Framework — SEP-2133 Primary, `_meta` Fallback

**Decision:** Primary MCP binding via SEP-2133 extension registration (vendor-prefixed, init-negotiated, mandatory-enforceable). `_meta` conventions become the backward-compatible fallback for servers not yet on SEP-2133.

**Context:** SEP-1724 and SEP-1502 referenced in prior research notes were confirmed as hallucinated by a sub-agent. Neither SEP exists. The actual MCP extension framework is **SEP-2133: Extensions** — Status: **Final** (not draft, not future). The MCP extension framework is shipped and operational. Verified at https://modelcontextprotocol.io/extensions/overview on April 23, 2026.

**What SEP-2133 provides (verified):**
1. Vendor-prefixed identifiers: `{vendor-prefix}/{extension-name}` format. Third parties use reversed domain names (e.g., `org.finos/afix-agent-card`).
2. Bilateral capability negotiation at initialization.
3. Mandatory enforcement via rejection — a server requiring a specific extension can reject non-compliant clients.
4. Graceful degradation — if one side supports an extension but the other does not, the supporting side falls back to core behavior.
5. Independent evolution — extensions evolve independently of core protocol.

**Impact on asymmetry analysis:** The original framing described MCP as having no protocol-level enforcement mechanism, relying entirely on `_meta` conventions and gateway configuration. That was wrong. SEP-2133 provides connection-level enforcement equivalent to A2A. The remaining asymmetry is narrower: per-request propagation of governance metadata through multi-hop chains within a session — not whether MCP can enforce governance (it can).

**Revised asymmetry:** A2A enforces governance per-message (every message carries extensions structurally). MCP enforces governance per-connection (SEP-2133 negotiation at init) but relies on the orchestrator to propagate per-request. The analogy: A2A is like a typed language where every function call carries its type annotations. MCP is like a language with typed module imports but untyped internal function calls.

**Alternatives considered:** `_meta` as primary with SEP-2133 as upgrade path. Rejected: SEP-2133 is shipped and Final; treating it as a future upgrade path misrepresents the current state of the protocol.

---

### Decision 6: ISO 42001 Elevation

**Decision:** Elevate ISO 42001 from footnote-level regulatory reference to named counterpart in the article's regulatory story. Three-part framing: EU AI Act creates the obligation, ISO 42001 creates the certification mechanism, AFIX creates the cross-enterprise interchange that makes certification status machine-readable at the point of agent interaction.

**Context:** ISO 42001 (published December 2023) is a management system standard certifying an organization's *processes* for responsible AI, not specific AI products.

**Schema impact:**
- `governance_certification_level` in Liability Metadata now maps tiers to ISO 42001: TIER_3_FULL = ISO 42001 certified + domain audit, TIER_2_STANDARD = ISO 42001 or NIST AI RMF aligned, TIER_1_BASIC = documented policy without certification, UNCERTIFIED = no formal posture.
- FS Agent Card `compliance_certifications.framework` enum already included `ISO_42001` — no change needed.

**Why this strengthens the argument:** ISO 42001 is intra-enterprise (certifies your firm's AI governance). AFIX is inter-enterprise (how certified firms communicate governance posture to each other). If ISO 42001 becomes the gold standard, `governance_certification_level` has something concrete to reference.

---

### Decision 7: FS Agent Card — Supply Chain Block

**Decision:** Add `ai_bom_reference` and `model_provenance` fields to FS Agent Cards. Frame as: "Your FS Agent Card tells the counterparty what the agent is authorized to do. The AI-BOM reference tells them what the agent is built from."

**Context:** Confirmed after reviewing Mend.io's AI Security Governance framework, which covers AI Bill of Materials (model version, training data, dependencies, vulnerability status) as a supply chain governance artifact. The `ai_bom_reference` field is a URI pointing to a published AI-BOM. Lightweight — a single URI field, not a new schema component.

**Reasoning:** Cross-enterprise interactions make model provenance critical — you are not just trusting the agent's authorized actions, you are trusting the model behind it. This connects intra-enterprise supply chain governance to cross-enterprise governance discovery.

**Field placement decision (open for stress-test):** Both discovery-time (Agent Card) and execution-time (Compliance Events) placement were considered. Agent Card placement confirmed for `ai_bom_reference` (static, deployment-time metadata). Compliance Events placement is appropriate for runtime attestations if they develop.

---

## Open Question Decisions

### OQ1: Liability Adjudication Without Central Authority — PHASED MODEL

**Decision:** Hybrid phased approach mapping to adoption maturity (see Decision 3 above for full phased model).

**Article framing:** Present the design space honestly, recommend the phased approach. Lead with the ISDA analog — the audience lives in the ISDA world.

---

### OQ2: Governance Observer Access to MCP Gateway Logs — TWO-TIER MODEL

**Decision:** Two-tier governance observation architecture.
- **Tier 1 (internal):** Splunk-style dashboards. Firms log AFIX Compliance Events to existing observability stacks alongside FIX drop copies. Configuration change, not new infrastructure.
- **Tier 2 (external):** Aggregator model (CorpAxe/Commcise precedent from MiFID II). Neutral third party collects standardized Compliance Events (not raw gateway logs), correlates by `chain_id`, redistributes across firms.

**Key insight:** The Observer does not need gateway logs. It needs Compliance Events — the *output* of gateway enforcement, not the mechanism. Gateway logs stay internal (Tier 1). Governance events flow to the aggregator (Tier 2).

**Article framing:** Pose the aggregator as an open market need. Flag the competitive dynamics wrinkle as an honest open question: would Bloomberg/FactSet/LSEG send governance data to an aggregator? Precedent exists (they use CorpAxe for interaction data), but agent governance metadata could be viewed as core IP. Do not prescribe the answer.

---

### OQ3: `chain_id` Correlation Without Central Registry — SOLVED BY DESIGN

See Decision 4 above. LEI:chain_id:hop_index composite key. No central registry required.

---

### OQ4: MCP Multi-Hop `_meta` Propagation — NARROWER PROBLEM THAN ORIGINALLY FRAMED

**Decision:** Two-phase mitigation. The problem is narrower than originally framed: it is not "MCP cannot enforce governance" (it can, via SEP-2133). It is specifically about mid-chain propagation in orchestrated multi-hop flows.

**Two mitigation layers:**
- **Phase 1 — Gateway-level enforcement with retry feedback.** Kong/Apigee gateway validates that every outbound MCP request includes required governance metadata. If missing, the gateway rejects AND returns a structured error specifying which fields are absent, allowing the requesting agent to retry with governance data included.
- **Phase 2 — Orchestrator-level convention.** We propose that agent framework authors (LangChain, CrewAI, Anthropic, etc.) adopt a convention: governance metadata negotiated via SEP-2133 extensions at session init must be treated as mandatory-propagate state for all tool calls within that session. This is a proposed convention, not an existing standard.

---

### OQ5: Entitlement Revocation Propagation — DUAL-MODE WITH OPTIONAL AGGREGATOR BROADCAST

**Decision:** Dual-mode enforcement for the pilot.
- **Eager mode (target: <30s propagation).** Push-based cache invalidation via gateway Admin API. The entitlement management system actively pushes invalidation events to the gateway when a revocation occurs.
- **Lazy mode (target: <60s propagation).** TTL-based with a 60-second maximum cache window. Acceptable for lower-risk entitlement changes. Simpler infrastructure.

**Cross-firm propagation:** The Tier 2 aggregator from OQ2 could serve a dual role as a revocation broadcast channel. This is additional scope but architecturally clean.

**Session-level enforcement (acknowledged, not recommended as default):** With SEP-2133, a revocation could trigger protocol-level session termination with a specific error code. Deterministic but too aggressive for the general case. Appropriate only for critical security revocations.

**Article framing:** Present as an engineering trade-off, not a solved problem. The TLS OCSP stapling analogy works: the industry spent years debating push vs. pull for certificate revocation and landed on "both, depending on context."

---

### OQ6: Article Tone — ESTABLISH → DEMONSTRATE → INVITE

**Decision:** Three-part tonal arc.

| Phase | Sections | Register | Mood |
|-------|----------|----------|------|
| **Establish** | 1, 2, 3 | Pure research authority | Indicative — sourced claims, no proposals |
| **Demonstrate** | 4, 5, 6 | Research + labeled proposals | EXISTS/WE PROPOSE discipline — every proposal clearly marked |
| **Invite** | 7 | Controlled earnestness | Warmth reserved for genuine intellectual conviction |

The arc earns the right to make an earnest appeal by proving analytical rigor first. Sections 1–3 are pure evidence. Sections 4–6 introduce proposals but label them explicitly. Section 7 is the only place warmth enters.

**GitHub repository as the bridge:** Full AFIX schemas (all 5 components), A2A and MCP binding specifications, ACGP pilot design with success criteria. These are the artifacts that convert "interesting paper" into "I can build from this."

---

## SEP Correction Record

**What happened:** Prior research notes cited "SEP-1724" (formal extension framework) and "SEP-1502" (extension directory structure) as draft proposals for MCP's extension mechanism. Neither SEP exists. Both numbers were hallucinated by a sub-agent.

**What actually exists:** **SEP-2133: Extensions** — Status: **Final** (not draft, not future). Verified at https://modelcontextprotocol.io/extensions/overview on April 23, 2026.

**Files corrected:** All research notes from the B.2.1 and B.2.2 workstreams. SEP-1724 and SEP-1502 references replaced with SEP-2133 throughout. Asymmetry analysis updated to reflect SEP-2133 as existing (not future). MCP binding redesigned: SEP-2133 primary, `_meta` fallback.

**Status:** Corrections completed. All research notes verified clean.
