# External Paper Comparison: Mend.io AI Security Governance Guide
_Date: 2026-04-23 | Status: Final_

---

## Source

Mend.io, "AI Security Governance: A Practical Framework for Security and Development Teams," 2026 (18-page PDF). Targets AppSec leads, CISOs, engineering managers. Covers: AI asset inventory, risk classification (5-dimension scoring), access controls, supply chain security (AI-BOM), monitoring/incident response, policy, and a 4-stage maturity model (Emerging → Developing → Controlling → Leading). Maps to NIST AI RMF, EU AI Act, ISO 42001, US Executive Order on AI.

**Tier assessment:** Tier 2 — vendor white paper (Mend.io is an AppSec company), but well-sourced and operationally grounded. Not peer-reviewed. Useful as market evidence for practitioner demand, not as an authoritative governance standard.

---

## Where It Sits Relative to AFIX

Mend.io covers **intra-enterprise governance Layers 2-3** with a **security lens** (access controls, supply chain, monitoring). Post 1 in this series covers Layers 1-4 with a governance-authority lens (who authorized the action, not just was the system secure). Post 2 covers Layer 5 (cross-enterprise governance via AFIX), which is entirely outside Mend.io's scope.

**Key validation:** Mend.io confirms the intra-enterprise governance gap is a recognized, live problem with practitioner demand. It does not touch the cross-enterprise case, which is the original territory of this research.

**Key distinction:** Mend.io asks "Is this AI system secure?" The AFIX research asks "Who authorized this agent to take this action, and can you prove it?" Their risk scoring captures Decision Authority as a classification dimension but does not solve it operationally. AFIX builds the governance primitives (authority types, commitment boundaries, action risk taxonomy) that operationalize it.

---

## Confirmed Decisions

### Decision 1: Supply Chain Dimension in Action Risk Taxonomy — ADOPT

**Finding:** Mend.io's fifth risk scoring dimension — Supply Chain Origin (reputable vendor / maintained open-source / unknown origin) — is absent from AFIX's action risk taxonomy.

**Decision:** Adopt. An action taken by a well-governed agent using an unvetted model is still risky. Model provenance should factor into action risk classification.

**Action for schema:** Added `model_provenance` and `supply_chain_attestation` fields to FS Agent Cards. Cross-enterprise interactions make this more critical — you are not just trusting the agent's authorized actions, you are trusting the model behind it. This connects Mend.io's AI-BOM concept to the governance discovery layer.

**Stress-test finding (for follow-up):** Supply chain provenance belongs in FS Agent Cards at discovery-time (for initial counterparty evaluation) and potentially in Compliance Events at execution-time (for runtime attestation). Design exercise pending.

---

### Decision 2: AI-BOM Reference in FS Agent Cards — ADOPT

**Finding:** Mend.io's AI Bill of Materials (model version, training data, dependencies, vulnerability status) is a concrete supply chain artifact that FS Agent Cards did not originally reference.

**Decision:** Adopt. Added `ai_bom_reference` field to FS Agent Cards — an optional URI pointing to a published AI-BOM. Connects intra-enterprise supply chain governance (Mend.io's territory) to cross-enterprise governance discovery (AFIX territory).

**Framing:** "Your FS Agent Card tells the counterparty what the agent is authorized to do. The AI-BOM reference tells them what the agent is built from." Lightweight — a single URI field, not a new schema component.

**Verification needed:** Existing AI-BOM standards (NIST AI RMF references, CycloneDX ML-BOM, SPDX AI) should be reviewed for URI-referenceable format compatibility. If no standard provides a URI-referenceable format, this gap should be noted.

---

### Decision 3: No Cross-Enterprise Validation from Mend.io — ACKNOWLEDGED

**Finding:** Mend.io confirms intra-enterprise governance demand but does not touch cross-enterprise governance. Post 2's thesis is neither validated nor challenged by this paper.

**Decision:** Acknowledged. No action needed. The research remains in first-principles territory for cross-enterprise governance, which is the value of the contribution.

---

### Decision 4: Timeline Reality — REFRAME, DO NOT SOFTEN

**Finding:** If most organizations are at Stage 1-2 for intra-enterprise governance, the preconditions for cross-enterprise governance (mature Layers 1-3 within each firm) seem distant.

**Context:** Mend.io is cross-industry, not financial-services-specific. JPMC, Morgan Stanley, BlackRock, and Bloomberg have active AI governance programs in place now. The real gap is not absence of governance — it is absence of *coordination*. Organizations are building governance programs in parallel but not collaborating. Different frameworks, no common language for cross-enterprise interactions.

**Decision:** Reframe. The article should hit this point directly: the problem is not absence of governance — it is absence of coordination. Organizations are building governance programs in parallel but not collaborating. AFIX does not replace what they are building internally; it gives them a common language for cross-enterprise interactions.

**Additional framing:** Timelines are compressing with AI. What used to take a year takes 6 months. The FIX/RIXML historical timelines (2-16 years) are directional, not predictive — AI-era consortium formation will be faster.

**Research completed:** The "independent governance, minimal collaboration" claim was verified using publicly available data. The refined framing: every major firm has active internal governance (JPMC, Morgan Stanley, BlackRock, Bloomberg). Cross-firm collaboration IS happening — but on governance *frameworks* (L1-L3 of the onion). Zero coordination exists on governance *interchange* (L4): when governed agents call each other, there is no shared vocabulary. Notably, JPMC, Bloomberg, and BlackRock are NOT listed as FINOS AI governance participants. The biggest AI investors are building independently.

---

### Decision 5: MCP Binding as Philosophical Weak Link — ADDRESS HEAD-ON

**Finding:** Mend.io's principle ("enforcement through tooling, not memory") exposes that MCP's `_meta` binding is closer to "policy enforced by memory" — gateway configuration can drift, orchestrators can drop `_meta` on multi-hop.

**Decision:** Address head-on in the article. Do not bury it. The asymmetry is the original contribution — owning the weakness makes the analysis credible. Frame: "A2A enforces governance structurally (the protocol rejects non-compliant messages). MCP enforces governance operationally (the gateway must be configured correctly). This is a real difference. It is also the same difference between compiled type safety and runtime validation — both work, one fails louder."

**SEP-2133 update to this framing:** With SEP-2133 confirmed as Final, the compiled/runtime analogy is more precise: MCP now has connection-level type safety (server rejects non-compliant clients at init). The remaining gap is per-request propagation in multi-hop flows. The analogy becomes "connection-level type safety with runtime propagation responsibility" rather than "purely runtime validation." (See Design Decisions, Decision 5.)

**Action:** The MCP multi-hop `_meta` drop is the strongest attack surface. The article presents it as an open engineering problem, not a solved one. SEP-2133 provides connection-level enforcement. Gateway enforcement is the interim for per-request validation. The orchestrator propagation convention is the near-term solution.

---

### Decision 6: Regulatory Alignment Table — ADOPT WITH VERIFICATION

**Finding:** Mend.io maps their framework to NIST AI RMF, EU AI Act, ISO 42001, US Executive Order. Post 2 has no equivalent for AFIX.

**Decision:** Adopt. Create a table mapping AFIX components to existing regulatory requirements. Every mapping must be verified against primary sources — no inferred mappings, no hallucinated regulatory language.

**Format decision:** Produce as a GitHub repository appendix (not inline — the article is already dense). Map each AFIX component to: NIST AI RMF functions (Govern, Map, Measure, Manage), EU AI Act risk tier obligations, ISO 42001 management system requirements, FINOS AI Governance Framework v2.0 controls.

**Quality bar:** Each mapping must cite the specific section/article/control number from the regulatory document, not a secondary summary. Flag any mapping where the connection is interpretive rather than explicit. This is a sub-agent task requiring primary source verification only.

**Status:** Not started. Deferred to a dedicated research sub-agent.

---

## New Research Tasks Generated

| # | Task | Priority | Source |
|---|------|----------|--------|
| 1 | Verify "independent governance, minimal collaboration" claim for FS firms | High | Decision 4 |
| 2 | Research AI-BOM standards (CycloneDX ML-BOM, SPDX AI) for URI-referenceable format | Medium | Decision 2 |
| 3 | Design exercise: `model_provenance` field placement (discovery-time vs. execution-time) | Medium | Decision 1 |
| 4 | Regulatory alignment table — AFIX ↔ NIST/EU AI Act/ISO 42001/FINOS (primary sources only) | Medium | Decision 6 |
| 5 | Test JSON Schema 2020-12 compatibility across major LLM providers (Gemini, OpenAI, Anthropic) | Low | Provider-level schema support varies |

---

## Methodology Note

The comparison methodology used here reflects how external papers should be evaluated against an in-progress research proposal:

1. **Scope placement first.** Identify where the external paper sits relative to the research scope. Mend.io covers Layers 2-3 (intra-enterprise, security lens). AFIX covers Layer 5 (cross-enterprise, governance interchange lens). The layers do not compete; they complement.

2. **Extract gotchas, not confirmations.** The value of external paper comparison is finding what the external paper has that the internal research missed — not confirming what both already know. The five "gotchas" above are all areas where Mend.io's operational grounding surfaced gaps in the AFIX schema design.

3. **Decide, do not hedge.** Each gotcha has a decision: ADOPT, ACKNOWLEDGE, REFRAME, or ADDRESS. Hedging ("worth considering") is not a decision. Every finding gets a disposition.

4. **Verify before incorporating.** Mend.io's AI-BOM framework references real standards (CycloneDX, SPDX). Before adding `ai_bom_reference` to FS Agent Cards, the verification step (Decision 2) must confirm that those standards produce URI-referenceable artifacts. If they do not, the field design needs adjustment.
