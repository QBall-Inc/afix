# Stress-Test Phase A: Assumptions
_Date: 2026-04-25 | Status: Final_

Five parallel stress-tests of the core assumptions underlying the AFIX binding specifications (revised for SEP-2133). Results confirmed: changes #1 and #3 require schema or framing updates; #2, #4, #5 are language-level adjustments.

---

## Assumption 1: Gateway Enforcement ≈ Protocol Enforcement

**Claim:** "Same outcome, different enforcement locus" — A2A and MCP produce equivalent governance despite different enforcement mechanisms.

**Verdict: PARTIALLY UNSUPPORTED**

Multi-hop propagation without per-message enforcement is a known-hard problem (W3C Trace Context analogy). Connection-level enforcement is not equivalent to per-message enforcement.

**Strongest counter:** Gateway misconfiguration is a real and common failure mode. Protocol enforcement (A2A's `required: true`) is structural — it cannot be misconfigured away. Gateway enforcement (MCP) can be disabled, bypassed, or incorrectly configured. Functional equivalence depends on operational competence, not architectural guarantees.

**Test design:** Deliberately misconfigure a Kong gateway (remove AFIX validation rule). Does governance fail silently? Compare: deliberately misconfigure an A2A agent (remove AFIX extension). Does governance fail loudly? The expected asymmetry: A2A fails structurally at the protocol level; MCP fails silently at runtime.

**Article fix:** Section 5 reframes the two-phase mitigation (gateway + orchestrator convention) with honest tone: this is a "structural limitation of the protocol pair," not a "manageable engineering problem." The analogy shifts from "compiled type safety vs. runtime validation" (which overstates MCP's weakness with SEP-2133 in the picture) to "connection-level type safety with runtime propagation responsibility."

**Status:** Language adjustment confirmed.

---

## Assumption 2: The Asymmetry Is Presentable Rather Than Disqualifying

**Claim:** The article frames the A2A/MCP asymmetry as a finding, not a problem.

**Verdict: PARTIALLY HOLDS — with material vulnerability**

**Strongest attack:** MCP is the MORE important binding in financial services (97M+ downloads, Bloomberg/FactSet/LSEG already deployed). A2A is the richer governance architecture but has minimal finserv deployment. The attack reads: "Your governance schema works best on the protocol FS firms aren't using yet."

**Counter:** MCP has the deployed infrastructure. A2A has the governance architecture. The dual-binding is the pragmatic answer precisely because the market has not converged. Mandating A2A ignores market reality; MCP-only governance accepts a structural limitation. Presenting both honestly is more credible than either advocacy.

**Material vulnerability acknowledged:** The asymmetry argument must lead with MCP (where readers start) and present A2A as the fuller implementation, framing the gap as a convergence trajectory rather than a permanent deficiency.

**Article fix:** Section 5 leads with MCP binding (where readers start), shows A2A as fuller implementation, frames asymmetry as convergence trajectory.

**Status:** Narrative reorder in Section 5 confirmed.

---

## Assumption 3: `ap2_mandate_ref` Does Not Violate Protocol Agnosticism

**Claim:** `ap2_mandate_ref` is an optional hash — an opaque reference. AFIX does not parse or depend on AP2.

**Verdict: DOES NOT HOLD**

A2A-specific field in the base schema violates protocol agnosticism per IETF/W3C standards principles, even as an optional opaque hash.

**Strongest counter:** A purist would argue that any reference to a specific external protocol creates conceptual coupling, even if technically opaque. If AP2 gets a field, why not TAP, UCP, SWIFT gpi? The slippery slope is real: an optional field today becomes a de facto dependency when tooling builds around it.

**Test design:** Remove `ap2_mandate_ref` entirely. Can the pilot still work? Yes — timestamp correlation replaces hash correlation, but with lower confidence. The field improves traceability but is not structurally required. This confirms the field is an enhancement, not a necessity, which means it belongs in a domain extension, not the base schema.

**Fix:** Move `ap2_mandate_ref` from base FS Agent Card schema to the `ext/agentic-commerce` domain extension using the existing Capability/Extension/Profile pattern. MCP-only deployments never see it. Cross-protocol traceability is preserved via `authorization._ext_refs`. Article treatment in Section 4: brief honest callout — "We caught this during stress-testing and refactored." Strengthens credibility.

**Status:** Schema update required before drafting. Confirmed.

---

## Assumption 4: The Six-Field Audit Minimum Is Truly Universal

**Claim:** FIX, FHIR, and RIXML converge on six fields — empirical evidence of a universal audit primitive.

**Verdict: PARTIAL — survives as interoperability minimum, NOT as regulatory compliance minimum**

**Strongest counter:** Three data points is pattern-spotting, not proof. Other audit standards (SOC 2 Common Criteria, ISO 27001 Annex A, NIST SP 800-53 AU controls) may define different minimal field sets. The "convergence" could be cherry-picked.

**Verification:** Checked against SOC 2 and NIST audit control frameworks. The six-field core survives as the interoperability minimum. But regulatory compliance requires additional fields:
- `model_id` + `dataset_version` — EU AI Act Article 12 logging requirements
- Microsecond timestamps — MiFID II RTS 25 (clock synchronization requirements for high-frequency trading venues)
- `log_integrity_hash` — EU AI Act Article 12 + SEC Rule 17a-4 tamper-evidence requirements

**Fix:** Two-tier model: "Six fields = interoperability schema, nine fields + constraints = regulatory compliance schema." Core schema in article body; compliance extension referenced in the regulatory story.

**Status:** Framing update confirmed.

---

## Assumption 5: EMV-Inspired Liability Shift Creates Adoption Incentive

**Claim:** Firms that invest in governance certification absorb less liability — the market incentive that drove chip card adoption.

**Verdict: PARTIAL — Phase 1 (bilateral) credible; Phase 2 (consortium) has a critical "who convenes?" gap**

**Strongest counter:** EMV liability shift worked because one party (the card network) had the authority to impose it. In a voluntary standards body (FINOS/CEAG), who imposes the shift? Without enforcement authority, the incentive is theoretical. The FINOS working group cannot mandate liability allocation. It can define the metadata format. Actual liability allocation is contractual (Agent Interaction Agreement) or regulatory.

**The critical gap for Phase 2:** EMV's consortium-level governance relied on Visa/Mastercard as conveners with real economic authority over their networks. No equivalent authority exists for cross-enterprise agent governance. The consortium model (Phase 2) requires someone to convene — and the blueprint does not identify who.

**Fix:** ISDA as primary analogy for Phase 1 (bilateral master agreements — the target audience lives in the ISDA world and will recognize the pattern immediately). Basel peer-monitoring for Phase 2. EMV becomes secondary reference for the proportional shift *mechanism* only, not the enforcement model.

**Article impact:** Language shift in Sections 5 and 6. Phased model unchanged; lead analogy changes. Present honestly: the schema enables the incentive; adoption requires either bilateral agreement or regulatory mandate to make it real.

**Status:** Framing update confirmed. ISDA leads, EMV is the mechanism reference.

---

## Summary Table

| # | Assumption | Verdict | Change Required |
|---|-----------|---------|-----------------|
| 1 | Gateway ≈ protocol enforcement | PARTIALLY UNSUPPORTED | Language: reframe as structural limitation |
| 2 | Asymmetry is presentable | PARTIALLY HOLDS | Narrative: lead with MCP, show A2A as fuller |
| 3 | `ap2_mandate_ref` protocol-agnostic | DOES NOT HOLD | Schema: move to domain extension |
| 4 | Six-field audit universal | PARTIAL | Framing: two-tier model (interop vs. compliance) |
| 5 | EMV liability incentive | PARTIAL | Framing: ISDA primary, EMV mechanism only |
