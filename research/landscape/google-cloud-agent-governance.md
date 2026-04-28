# Source Brief: Google Cloud Agent Governance — Gemini Enterprise Agent Platform
_Date: 2026-04-26 | Status: Final_

---

## Source Details

**Source 1:** "Introducing Gemini Enterprise Agent Platform." Google Cloud Blog, April 22, 2026.
- URL: https://cloud.google.com/blog/products/ai-machine-learning/introducing-gemini-enterprise-agent-platform
- [Tier 1] Primary vendor documentation.

**Source 2:** "Agent Gateway overview." Google Cloud Documentation, April 2026.
- URL: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/agent-gateway-overview
- [Tier 1] Primary vendor documentation.

**Source 3:** "Govern your agents." Google Cloud Documentation, April 2026.
- URL: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern
- [Tier 1] Primary vendor documentation.

**Source 4:** A2A Protocol Specification v1.2. a2a-protocol.org, April 2026.
- URL: https://a2a-protocol.org/latest/specification/
- [Tier 1] Official protocol specification.

**Source 5:** "Google Cloud Next 2026: The Agentic Enterprise Control Plane Comes into View." Bain & Company, April 2026.
- URL: https://www.bain.com/insights/google_cloud_next_2026_the_agentic_enterprise_control_plane_comes_into_view/
- [Tier 2] Analyst report.

---

## Key Findings

Google launched three governance primitives as part of Gemini Enterprise Agent Platform (the Vertex AI rebrand) at Cloud Next 2026.

**Agent Identity.** Every agent gets a unique cryptographic ID used as the principal for authorization decisions. Creates auditable trail mapped to authorization policies. Integrates with OAuth 2.0 via Agent Identity Auth Manager.¹²

**Agent Registry.** Centralized catalog indexing every agent, tool, MCP server, and skill across an organization. Single source of truth for discovery and governance posture.¹²

**Agent Gateway.** The enforcement point. Inspects both A2A and MCP traffic natively — not a transport-agnostic proxy but a protocol-aware inspection layer. Enforces least-privilege at runtime. Handles mTLS automatically. Integrates Model Armor for prompt injection/data leakage protection. Symantec DLP integration for real-time inspection.¹²

**A2A v1.2 Signed Agent Cards.** Cryptographic signatures (JWS) over JSON Canonicalization Scheme, enabling a receiving agent to verify the card was issued by the domain owner. A2A is now governed by Linux Foundation's Agentic AI Foundation with 150+ member organizations. Production deployments at Microsoft, AWS, Salesforce, SAP, ServiceNow.⁴

**Architecture:** Gateway-centric, protocol-aware, platform-native. The stack is: Agent Identity (who) → Agent Registry (what exists) → Agent Gateway (what's allowed) → Model Armor (what's safe). Structurally identical to Cloudflare's approach (gateway-centric, platform-monolithic) except Google covers both A2A and MCP in a single enforcement surface.

---

## Source Concordance

**Agreement:**
- Confirms gateway-centric enforcement pattern — now validated for both A2A and MCP by the A2A protocol's own creator
- Validates independent governance middleware construction by infrastructure providers
- A2A Signed Agent Cards strengthen agent governance binding — JWS-signed cards with domain verification are the identity primitive that governance interchange schemas extend
- Agent Gateway validates the "self-documenting" A2A governance thesis — governance metadata embedded in A2A messages would be visible to Agent Gateway without modification

**Divergence:**
- None. Google's architecture is complementary to open interchange standards, not competitive.

**Gaps — what Google does NOT address:**
- No governance interchange schema for cross-enterprise metadata exchange
- No multi-hop governance chain tracking
- No liability metadata or attribution
- No regulatory jurisdiction interchange
- No entitlement interchange beyond platform-level access control
- No compliance event schema beyond operational audit logs
- Governance is entirely Google Cloud-native — Agent Gateway enforces policies on traffic within a single organization's control plane

---

## Analyst Take

Google shipped the strongest governance stack to date — identity, registry, gateway, audit — and it is entirely intra-enterprise. The fact that the protocol's own creator does not govern across organizational boundaries is the strongest possible evidence for why open governance interchange standards are needed. Google is the enforcement infrastructure that governance metadata would flow through, not a competing interchange standard.

The Cloudflare comparison sharpens: Cloudflare built MCP-only gateway governance. Google built A2A+MCP gateway governance. Neither built interchange. The framing holds: the two largest gateway providers have independently converged on the same architectural boundary — enforcement stops at the org perimeter.

Counter: Google could extend Agent Gateway to propagate governance context in A2A messages, making an open interchange standard redundant for A2A. But they have not, and their commercial incentive is to deepen platform lock-in, not enable portable governance.

---

## Sources

1. [Tier 1] "Introducing Gemini Enterprise Agent Platform." Google Cloud Blog, 2026-04-22. https://cloud.google.com/blog/products/ai-machine-learning/introducing-gemini-enterprise-agent-platform
2. [Tier 1] "Agent Gateway overview." Google Cloud Documentation, 2026-04. https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/agent-gateway-overview
3. [Tier 1] "Govern your agents." Google Cloud Documentation, 2026-04. https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern
4. [Tier 1] A2A Protocol Specification v1.2. a2a-protocol.org, 2026-04. https://a2a-protocol.org/latest/specification/
5. [Tier 2] "Google Cloud Next 2026." Bain & Company, 2026-04. https://www.bain.com/insights/google_cloud_next_2026_the_agentic_enterprise_control_plane_comes_into_view/
