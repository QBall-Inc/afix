# AFIX MCP Protocol Binding

**Version:** 1.0.0
**Protocol:** Anthropic Model Context Protocol (MCP) — [Specification](https://spec.modelcontextprotocol.io/)
**Extension Framework:** SEP-2133 (MCP Server Extension Proposal) — [Draft](https://github.com/modelcontextprotocol/specification/discussions/2133)
**Status:** Draft — published for community review

---

## Overview

This document defines how AFIX governance primitives attach to the Model Context Protocol. Three attachment mechanisms are specified, in order of preference:

1. **SEP-2133 Extension Framework** (primary) — bilateral capability negotiation at initialization; server rejects non-compliant clients.
2. **`_meta` Conventions** (fallback) — for servers not yet implementing SEP-2133; governance metadata travels in the `_meta` field on requests and results.
3. **Gateway Enforcement** (infrastructure) — for environments where server-side modification is not feasible; governance is enforced at the proxy layer.

All three mechanisms produce equivalent governance semantics. The attachment layer is a deployment decision, not a schema decision.

---

## Primary: SEP-2133 Extension Framework

### Capability Registration

AFIX registers under the `finos.org/afix-governance` capability namespace. All AFIX fields use vendor-prefixed identifiers to prevent collision with other MCP extensions.

**Server capability declaration (initialization response):**

```json
{
  "capabilities": {
    "extensions": {
      "finos.org/afix-governance": {
        "version": "1.0.0",
        "components": {
          "agentCard": { "required": true },
          "complianceEvents": { "required": true },
          "entitlementInterchange": { "required": true },
          "liabilityMetadata": { "required": true },
          "governanceHopRecord": { "required": false }
        },
        "enforcement": "strict"
      }
    }
  }
}
```

**Client capability declaration (initialization request):**

```json
{
  "capabilities": {
    "extensions": {
      "finos.org/afix-governance": {
        "version": "1.0.0",
        "components": {
          "agentCard": true,
          "complianceEvents": true,
          "entitlementInterchange": true,
          "liabilityMetadata": true
        }
      }
    }
  }
}
```

### Negotiation and Rejection

Negotiation is bilateral and occurs at MCP session initialization, before any tool calls are permitted. The server evaluates the client's declared capabilities against its required components. If any required component is missing from the client declaration, the server must reject the session with a structured error.

**Rejection response (missing required component):**

```json
{
  "error": {
    "code": -32600,
    "message": "AFIX governance components required but not declared by client",
    "data": {
      "namespace": "finos.org/afix-governance",
      "missing": ["complianceEvents", "entitlementInterchange"]
    }
  }
}
```

Once negotiation succeeds, the agreed capability set applies to all subsequent requests in the session. The server may enforce governance invariants on every `CallToolRequest` and `CallToolResult`.

### Per-Request Metadata Attachment (SEP-2133 path)

With SEP-2133, governance metadata travels in the extension-scoped metadata block on `CallToolRequest` and `CallToolResult`:

```json
{
  "method": "tools/call",
  "params": {
    "name": "get_portfolio_positions",
    "arguments": { "portfolioId": "PORT-001" },
    "extensions": {
      "finos.org/afix-governance": {
        "entitlementInterchange": {
          "requestingAgentId": "agt_portfolio_analyzer_v2",
          "requestingFirmLEI": "5493001KJTIIGC8Y1R12",
          "dataAssets": [
            {
              "assetId": "bloomberg.equity.US.realtime",
              "permissionLevel": "READ",
              "licenseRef": "BBG-ENT-2024-00891",
              "expiresAt": "2025-12-31T23:59:59Z"
            }
          ],
          "redistributionAllowed": false
        },
        "liabilityMetadata": {
          "decisionId": "dec_9c3f1a7e",
          "agentChain": ["agt_orchestrator_v1", "agt_portfolio_analyzer_v2"],
          "humanInLoop": false,
          "recommendationType": "DATA_RETRIEVAL",
          "liabilityOwner": "agt_orchestrator_v1",
          "firmLiabilityAccepted": true
        }
      }
    }
  }
}
```

---

## Fallback: `_meta` Conventions

For MCP servers not yet implementing SEP-2133, AFIX metadata travels in the `_meta` field. This is a supported MCP mechanism for carrying non-protocol metadata and requires no server modification to the core MCP stack.

### Key Naming Convention

All AFIX `_meta` keys follow the pattern:

```
finos.org/afix/{component}/{field}
```

Where `{component}` is one of: `agent-card`, `compliance-events`, `entitlement`, `liability`, `governance-hop`.

### Request Attachment

```json
{
  "method": "tools/call",
  "params": {
    "name": "get_portfolio_positions",
    "arguments": { "portfolioId": "PORT-001" },
    "_meta": {
      "finos.org/afix/entitlement/requestingAgentId": "agt_portfolio_analyzer_v2",
      "finos.org/afix/entitlement/requestingFirmLEI": "5493001KJTIIGC8Y1R12",
      "finos.org/afix/entitlement/dataAssets": [
        {
          "assetId": "bloomberg.equity.US.realtime",
          "permissionLevel": "READ",
          "licenseRef": "BBG-ENT-2024-00891",
          "expiresAt": "2025-12-31T23:59:59Z"
        }
      ],
      "finos.org/afix/liability/decisionId": "dec_9c3f1a7e",
      "finos.org/afix/liability/liabilityOwner": "agt_orchestrator_v1",
      "finos.org/afix/liability/firmLiabilityAccepted": true
    }
  }
}
```

### Result Attachment

```json
{
  "result": {
    "content": [{ "type": "text", "text": "Portfolio positions retrieved." }],
    "_meta": {
      "finos.org/afix/compliance-events/events": [
        {
          "eventId": "evt_7f3a2b1c",
          "eventType": "DATA_ACCESS",
          "timestamp": "2025-03-15T14:23:01Z",
          "agentId": "agt_portfolio_analyzer_v2",
          "dataClassification": "MNPI",
          "approved": true
        }
      ]
    }
  }
}
```

### `_meta` Limitations

The `_meta` fallback does not support session-level negotiation or server-side rejection of non-compliant clients. Enforcement in the `_meta` path must occur at the gateway layer (see below) or via application logic. For production deployments requiring server-enforced governance, SEP-2133 is required.

---

## Gateway Enforcement

For environments where neither SEP-2133 nor `_meta` instrumentation is feasible at the server — for example, when calling third-party MCP servers — governance enforcement shifts to the proxy layer. Three patterns are documented.

### Pattern 1: Kong AI MCP Proxy

Kong's MCP proxy plugin intercepts `tools/call` requests before they reach the upstream server. An AFIX validation policy runs as a pre-plugin and attaches compliance events as a post-plugin.

**Conceptual Kong plugin configuration:**

```yaml
plugins:
  - name: afix-governance
    config:
      mode: enforce
      required_components:
        - entitlementInterchange
        - liabilityMetadata
      on_missing: reject          # alternatives: warn, passthrough
      audit_backend: splunk       # write compliance events to Splunk
      agent_card_cache_ttl: 300   # seconds
      high_risk_escalation: true  # enforce human-in-loop for HIGH risk tier
```

The plugin reads governance metadata from `_meta` (fallback) or SEP-2133 extension fields, validates entitlement expiry and liability closure, writes compliance events to the configured audit backend, and either forwards the request or returns a `403` with a structured error.

### Pattern 2: Apigee Policy Chain

Apigee enforces AFIX governance as a sequential policy chain on the MCP proxy flow.

**Conceptual policy chain:**

```xml
<PreFlow>
  <Request>
    <Step><Name>JsonFs-ExtractGovernanceMetadata</Name></Step>
    <Step><Name>JsonFs-ValidateEntitlementExpiry</Name></Step>
    <Step><Name>JsonFs-CheckLiabilityClosure</Name></Step>
    <Step><Name>JsonFs-EnforceHighRiskEscalation</Name></Step>
  </Request>
</PreFlow>
<PostFlow>
  <Response>
    <Step><Name>JsonFs-AttachComplianceEvents</Name></Step>
    <Step><Name>JsonFs-WriteAuditLog</Name></Step>
  </Response>
</PostFlow>
```

Each policy step is a standalone JavaScript or Java callout. Policies can be shared across API proxies, enabling consistent AFIX enforcement across all MCP-connected tools in an Apigee organization.

### Pattern 3: Custom Proxy (Bloomberg / FactSet / LSEG pattern)

Bloomberg, FactSet, and LSEG have each independently built MCP middleware that covers identity, access control, audit, and rate limiting. A custom proxy implementing AFIX sits in front of the existing middleware and translates between AFIX governance metadata and the vendor's internal governance format.

**Conceptual architecture:**

```
MCP Client
    |
    v
[AFIX Validation Proxy]
    |  — validates AFIX governance metadata
    |  — translates to vendor governance format
    v
[Vendor MCP Middleware]  ← existing Bloomberg/FactSet/LSEG stack
    |  — identity, access control, rate limiting
    v
[MCP Server / Data API]
```

This pattern allows AFIX adoption without replacing existing governance infrastructure. The proxy acts as a translation and validation layer, not a replacement.

### Pattern 4: Cloudflare MCP Server Portals

Cloudflare's MCP Server Portal is the most fully realized production implementation of the gateway pattern. The portal aggregates multiple upstream MCP servers behind a single endpoint, with governance enforcement at the network layer via Cloudflare Access and Cloudflare Gateway.

**Capabilities:**

- **Identity:** Cloudflare Access acts as OAuth provider. JWT assertion headers (`Cf-Access-Jwt-Assertion`) propagate identity to upstream servers. Two auth patterns: self-hosted (server validates JWT directly) and Access-for-SaaS (OIDC authorization code flow).
- **Tool-scope authorization:** Admins enable/disable individual tools per portal. Policies enforce along three axes: identity (user/group), conditions (device posture, location), and scope (specific tools).
- **DLP:** Gateway HTTP policies scan MCP portal traffic against predefined profiles (credentials, financial information). Policies target upstream server URLs.
- **Audit logging:** Portal logs capture timestamp, status, server name, capability (tool), and duration. Enterprise-tier Logpush enables SIEM export for long-term retention.
- **Shadow MCP detection:** Cloudflare Gateway inspects corporate internet traffic for unauthorized MCP usage via hostname matching, URL path matching, and JSON-RPC body inspection (detecting `tools/call`, `initialize`, `prompts/get` method fields).

**Conceptual architecture:**

```
MCP Client
    |
    v
[Cloudflare MCP Server Portal]
    |  — Cloudflare Access: OAuth / JWT identity
    |  — tool-scope allowlisting
    |  — DLP body inspection
    |  — audit logging (5-field telemetry)
    v
[Upstream MCP Server(s)]  ← Cloudflare-hosted or third-party
```

For AFIX, the portal model validates the gateway enforcement architecture. An AFIX-compliant deployment would inject governance metadata into `_meta` before the portal boundary — the portal's existing logging captures the richer governance record alongside its native telemetry. The portal's five-field audit log (operational telemetry) and AFIX's six-field Compliance Event (governance interchange) are complementary, not competing.

**Reference:** Goldberg, Carey, Anguiano. "Scaling MCP adoption." Cloudflare Blog, 2026-04-14.

---

## Agent Card Discovery

Two mechanisms are defined for discovering a server's FS Agent Card via MCP.

### Mechanism 1: Dedicated MCP Tool (Current)

Servers expose a `afix_agent_card` tool with no required arguments. Any MCP client can call this tool to retrieve the server's full FS Agent Card payload.

```json
{
  "name": "afix_agent_card",
  "description": "Returns the AFIX Agent Card for this MCP server, including regulatory jurisdictions, compliance frameworks, data classifications, risk tier, and approved counterparties.",
  "inputSchema": {
    "type": "object",
    "properties": {}
  }
}
```

**Example response:**

```json
{
  "content": [
    {
      "type": "text",
      "text": "{\"regulatoryJurisdictions\": [\"SEC\", \"FCA\"], \"riskTier\": \"HIGH\", \"complianceFrameworks\": [\"MiFID II\"], \"dataClassifications\": [\"MNPI\", \"PII\"]}"
    }
  ]
}
```

### Mechanism 2: `.well-known` Endpoint (Future)

A future revision of this binding will specify discovery via a `.well-known/afix-agent-card` HTTP endpoint, consistent with emerging MCP discovery conventions. This mechanism is reserved for a future version and is not part of this binding.

---

## Asymmetry with A2A Binding

The MCP binding differs from the A2A binding in three material ways:

**Per-request metadata only.** MCP has no concept of a persistent session-level task object equivalent to A2A's `Task`. Governance metadata must be re-asserted on every `CallToolRequest`. Implementations should cache entitlement and liability payloads at the client layer to avoid redundant payload construction, but the metadata itself cannot be elided.

**Multi-hop requires orchestrator propagation.** A2A's task model allows governance metadata to accumulate across hops on the task object. In MCP, each hop is an independent request-response pair. Multi-hop chains require the orchestrating agent to explicitly propagate governance metadata — including Governance Hop Records — on each downstream `CallToolRequest`. There is no automatic propagation.

**Audit trail is split.** In A2A, compliance events attach to message metadata and travel with the protocol payload end-to-end. In the MCP `_meta` path, compliance events on results may not survive all proxy and middleware layers. Gateway-enforced deployments must ensure the audit backend receives compliance events directly from the proxy rather than relying on them to survive in the protocol payload.

These asymmetries are inherent to MCP's stateless, per-request architecture. They are not defects in this binding — they are constraints to be managed in implementation.

---

## Sources

1. Model Context Protocol Specification — [https://spec.modelcontextprotocol.io/](https://spec.modelcontextprotocol.io/)
2. MCP `_meta` Field Definition — [https://spec.modelcontextprotocol.io/specification/basic/utilities/](https://spec.modelcontextprotocol.io/specification/basic/utilities/)
3. MCP `CallToolRequest` and `CallToolResult` Schema — [https://spec.modelcontextprotocol.io/specification/server/tools/](https://spec.modelcontextprotocol.io/specification/server/tools/)
4. SEP-2133: MCP Server Extension Proposal — [https://github.com/modelcontextprotocol/specification/discussions/2133](https://github.com/modelcontextprotocol/specification/discussions/2133)
5. MCP Initialization and Capability Negotiation — [https://spec.modelcontextprotocol.io/specification/basic/lifecycle/](https://spec.modelcontextprotocol.io/specification/basic/lifecycle/)
6. Kong AI Gateway MCP Support — [https://konghq.com/products/kong-ai-gateway](https://konghq.com/products/kong-ai-gateway)
7. Apigee API Management — [https://cloud.google.com/apigee](https://cloud.google.com/apigee)
8. FINOS Common Cloud Controls for AI (CC4AI) — [https://www.finos.org/common-cloud-controls](https://www.finos.org/common-cloud-controls)
9. Cloudflare Enterprise MCP Reference Architecture — [https://blog.cloudflare.com/enterprise-mcp/](https://blog.cloudflare.com/enterprise-mcp/)
10. Cloudflare MCP Governance Documentation — [https://developers.cloudflare.com/agents/model-context-protocol/governance/](https://developers.cloudflare.com/agents/model-context-protocol/governance/)
