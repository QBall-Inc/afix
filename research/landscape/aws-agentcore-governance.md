# Source Brief: AWS Bedrock AgentCore — Harness and Policy Governance
_Date: 2026-04-26 | Status: Final_

---

## Source Details

**Source 1:** "AgentCore harness [Preview]." AWS Documentation, accessed April 27, 2026.
- URL: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness.html
- [Tier 1] Primary vendor documentation.

**Source 2:** "Security and access controls." AWS Documentation, accessed April 27, 2026.
- URL: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-security.html
- [Tier 1] Primary vendor documentation.

**Source 3:** "Policy in Amazon Bedrock AgentCore." AWS Documentation, accessed April 27, 2026.
- URL: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy.html
- [Tier 1] Primary vendor documentation.

**Source 4:** "AgentCore Observability." AWS Documentation, accessed April 27, 2026.
- URL: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html
- [Tier 1] Primary vendor documentation.

**Source 5:** "Policy in Amazon Bedrock AgentCore is now generally available." AWS, March 3, 2026.
- URL: https://aws.amazon.com/about-aws/whats-new/2026/03/policy-amazon-bedrock-agentcore-generally-available/
- [Tier 1] Official GA announcement.

---

## Key Findings

Initial assessment that AgentCore harness "provides only observability" is incorrect. The harness is a **managed agent runtime** providing seven capability categories.¹²³⁴

**Genuine governance (enforcement):**
- **Cedar-based policy engine** on AgentCore Gateway — intercepts every agent-to-tool call, evaluates against deterministic rules before permitting access. Supports fine-grained controls on user identity and tool input parameters. GA since March 2026 across 13 regions.³⁵
- **Identity propagation** — inbound OAuth threads end-user identity through the agent to downstream tools. Per-user credential scoping, not shared service accounts.²
- **Isolated execution** — each session runs in its own Firecracker microVM. No shared state or filesystem between sessions.²
- **IAM execution roles** — least-privilege, ARN-scoped permissions on every harness API.²

**Observability (monitoring, not enforcement):**
- OTEL-compatible telemetry — traces, spans, metrics, logs stored in CloudWatch. Execution path visualization, error breakdowns, token usage tracking.⁴
- Audit logging of policy decisions via CloudWatch integration.³

**Architecture:** Three-layer: SDK/framework (Strands Agents, open-source) → managed runtime (Firecracker microVMs) → gateway (AgentCore Gateway with Cedar policy engine). Policy enforcement is gateway-centric — Cedar policies are associated with gateways, not individual agents. Direct tool invocations bypassing the gateway are not policy-controlled.³

The harness supports MCP servers as a tool connectivity option alongside the proprietary Gateway. No mention of A2A protocol support.¹

---

## Source Concordance

**Agreement:**
- Confirms gateway-centric enforcement pattern — independent governance middleware construction by major cloud providers
- Cedar-based policy engine is entirely AWS-scoped — proprietary policy language, no cross-enterprise portability
- A firm on AgentCore and a firm on Cloudflare have zero shared governance vocabulary
- MCP server support creates a natural integration surface for open governance standards via MCP bindings

**Divergence:**
- None. AWS's architecture is complementary to open interchange standards, not competitive.

**Gaps — what AWS does NOT address:**
- No governance metadata exchange between independent organizations
- No structured governance records that travel with agent interactions across enterprise boundaries
- No multi-party governance chain tracking
- No interoperable policy formats (Cedar is AWS-specific)
- Identity model limited to a single trust domain (one IdP per harness) — no federated governance identity across enterprises
- No A2A protocol support

---

## Analyst Take

AWS built exactly what you would expect — strong single-tenant governance with no interoperability story. The absence of cross-enterprise interchange in the largest cloud provider's agent governance stack is the strongest market-signal evidence for open governance interchange: every major platform is building governance walls, and nobody is building governance bridges.

AWS's Cedar policies, Cloudflare's Access JWT assertions, and Google's Agent Gateway policies are three incompatible governance vocabularies. When a Cedar-governed agent calls a Cloudflare-governed agent, neither side has a shared understanding of what "identity" means or what an "audit event" looks like. This is precisely the fragmentation that open interchange standards are designed to prevent.

Cedar is not a threat to open interchange standards; it is a potential binding target. A governance interchange binding could translate governance chain metadata into Cedar policy context, allowing AgentCore's enforcement engine to validate incoming cross-enterprise agent requests against propagated governance records.

Counter: AWS could add cross-account/cross-org governance propagation in a future release, but that would still be AWS-to-AWS, not AWS-to-Cloudflare-to-Google.

---

## Sources

1. [Tier 1] "AgentCore harness [Preview]." AWS Documentation, 2026-04-27. https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness.html
2. [Tier 1] "Security and access controls." AWS Documentation, 2026-04-27. https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-security.html
3. [Tier 1] "Policy in Amazon Bedrock AgentCore." AWS Documentation, 2026-04-27. https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy.html
4. [Tier 1] "AgentCore Observability." AWS Documentation, 2026-04-27. https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html
5. [Tier 1] "Policy in AgentCore GA." AWS, 2026-03-03. https://aws.amazon.com/about-aws/whats-new/2026/03/policy-amazon-bedrock-agentcore-generally-available/
