# Source Brief: Oracle Runtime Governance — From Model Safety to Runtime Governance
_Date: 2026-04-26 | Status: Final_

---

## Source Details

**Source 1:** "From Model Safety to Runtime Governance." Kishore Pusukuri. Oracle AI & Data Science Blog, April 23, 2026.
- URL: https://blogs.oracle.com/ai-and-datascience/runtime-governance-enterprise-agentic-ai
- [Tier 1] Official Oracle engineering blog, named author (Senior AI/Systems Leader; Oracle, Walmart, Apple background).

---

## Key Findings

Oracle defines runtime governance as shifting the unit of control from the model response to the **governed action trajectory** — the full execution path including proposed actions, identity bindings, delegated authority, resource consumption (budgets), state transitions, and side effects on enterprise data. The core question moves from "was the model response safe?" to "is the next specific action authorized under current policy, identity, approval state, data boundaries, and budget constraints?"¹

**Four-layer control model (OCI AI Governance Framework):**¹
- **L4 (Rules):** Risk appetite, risk tiers, promotion gates, approval workflows, policy lifecycle
- **L3 (Gating):** Registry-gated admissibility — which models, tools, connectors, corpora, regions, and identities may be used at all. Least-privilege enforcement
- **L2 (Behavior):** Runtime enforcement. The Agent Runtime Controller evaluates schema-valid proposed actions against policy packs, identity/approval bindings, and budget state before execution. Returns enforceable outcomes: ALLOW, ALLOW_WITH_REDACTION, REQUIRE_REVIEW, or DENY
- **L1 (Evidence):** Structured traces, provenance hashes, tamper-resistant audit trails, Structured Decision Records for replay and audit

L1 feeds back into L4 — evidence is an operating principle, not just an audit artifact. Three control moments: design time, promotion time, runtime.

**Threat model explicitly names:** unauthorized tool use, unsafe data egress, denial-of-wallet loops, excessive autonomy, supply-chain compromise, state/memory poisoning, cross-session leakage, confused deputy behavior, and overbroad delegated authorization.¹

**Architecture:** Application-layer enforcement via Agent Runtime Controller, not gateway-layer. The controller evaluates each structured proposed action against active policy packs before tool execution. This is a fundamentally different enforcement point from Cloudflare (network-layer gateway) and Google (protocol-aware gateway).

**No schemas published.** The "Governance Envelope" is described conceptually as a "machine-consumable execution contract" but no schema or specification is provided. No standards body references (no FINOS, NIST AI RMF, ISO 42001, A2A, or MCP governance standards).¹

---

## Source Concordance

**Agreement:**
- Confirms the runtime governance gap is real — pre-deployment governance (model cards, evals, risk assessments) is insufficient
- Validates that governance must operate at the per-action level during execution
- The intra-/cross-enterprise boundary is sharp: Oracle builds runtime governance within a single enterprise's control plane

**Divergence:**
- None. Oracle's scope is strictly intra-enterprise.

**Gaps — what Oracle does NOT address:**
- Entirely intra-enterprise — no cross-enterprise agent communication, governance interchange, or multi-party governance chains
- No A2A protocol awareness
- No schema definitions published (Governance Envelope is conceptual only)
- No standards body alignment
- No liability attribution across firm boundaries

---

## Analyst Take

Oracle's framework is significant validation. Their articulation of the runtime governance gap from a Tier 1 vendor strengthens the case for open governance interchange standards. Oracle's Governance Envelope is the internal enforcement mechanism; open interchange schemas (Compliance Events, Governance Hop Records) are how governance attestations travel across firm boundaries. These are complementary layers.

Schema convergence opportunity exists: Oracle's "schema-valid proposed action" requirement and "Structured Decision Records" map conceptually to Agent Cards and Compliance Events in emerging governance interchange proposals. Oracle's L1 Evidence layer would be a natural producer of cross-enterprise governance attestations — it generates exactly the structured records that an interchange standard could carry.

Enforcement point comparison: Cloudflare enforces at the gateway (network-layer). Oracle enforces at the runtime controller (application-layer). Google enforces at the protocol-aware gateway. Open interchange standards operate at the protocol layer (governance metadata traveling with the interaction). Four distinct enforcement points, all necessary, all complementary.

Counter: Oracle has no incentive to adopt an external interchange standard unless customers or regulators demand it.

---

## Sources

1. [Tier 1] Pusukuri, K. "From Model Safety to Runtime Governance." Oracle AI & Data Science Blog, 2026-04-23. https://blogs.oracle.com/ai-and-datascience/runtime-governance-enterprise-agentic-ai
