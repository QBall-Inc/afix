# AFIX A2A Protocol Binding

**Version:** 1.0.0
**Protocol:** Google Agent-to-Agent (A2A) — [Specification](https://google.github.io/A2A/)
**Status:** Draft — published for community review

---

## Overview

This document defines how AFIX governance primitives attach to the A2A protocol. AFIX uses A2A's native extension mechanism — no protocol modifications required. Extensions are declared on the Agent Card and negotiated at task initiation. Metadata attaches to messages and tasks per the rules below.

---

## Extension Registration

Five AFIX extensions are defined for A2A. Four are required for compliant AFIX agent interactions; the Governance Hop Record is optional and only required in multi-hop chains.

| Extension Name | URI | Type | Required |
|---|---|---|---|
| FS Agent Card | `https://finos.org/afix/agent-card/v1` | `object` | Yes |
| Compliance Events | `https://finos.org/afix/compliance-events/v1` | `array` | Yes |
| Entitlement Interchange | `https://finos.org/afix/entitlement/v1` | `object` | Yes |
| Liability Metadata | `https://finos.org/afix/liability/v1` | `object` | Yes |
| Governance Hop Record | `https://finos.org/afix/governance-hop/v1` | `array` | No (multi-hop only) |

---

## Agent Card Extension

The FS Agent Card attaches to an A2A Agent Card's `capabilities.extensions[]` array as a named extension with its payload in the `params` field. This allows standard A2A discovery tooling to surface AFIX metadata without modification.

```json
{
  "capabilities": {
    "extensions": [
      {
        "uri": "https://finos.org/afix/agent-card/v1",
        "required": true,
        "params": {
          "regulatoryJurisdictions": ["SEC", "FCA", "ESMA"],
          "complianceFrameworks": ["MiFID II", "SEC Rule 17a-4"],
          "dataClassifications": ["MNPI", "PII", "PUBLIC"],
          "riskTier": "HIGH",
          "humanEscalationRequired": true,
          "auditRetentionDays": 2555,
          "licenseType": "BROKER_DEALER",
          "approvedCounterparties": ["LEI:5493001KJTIIGC8Y1R12"]
        }
      }
    ]
  }
}
```

The full FS Agent Card schema is defined in [`/schemas/agent-card-v1.0.0.json`](/schemas/agent-card-v1.0.0.json).

---

## Extension Negotiation

AFIX uses the `X-A2A-Extensions` header for bilateral capability negotiation at the HTTP transport layer.

**Client declaration (request):**

```
X-A2A-Extensions: https://finos.org/afix/agent-card/v1,
                  https://finos.org/afix/compliance-events/v1,
                  https://finos.org/afix/entitlement/v1,
                  https://finos.org/afix/liability/v1
```

**Server validation and echo (response):**

The server validates that all required extensions are declared by the client. If any required extension is missing, the server must return `400 Bad Request` with a structured error identifying the missing extensions. If negotiation succeeds, the server echoes the activated extensions:

```
X-A2A-Extensions-Active: https://finos.org/afix/agent-card/v1,
                          https://finos.org/afix/compliance-events/v1,
                          https://finos.org/afix/entitlement/v1,
                          https://finos.org/afix/liability/v1
```

Extension negotiation occurs once per session. The negotiated set applies to all subsequent messages and tasks within that session.

**Error response (missing required extension):**

```json
{
  "error": {
    "code": -32600,
    "message": "Required AFIX extensions not declared",
    "data": {
      "missing": ["https://finos.org/afix/compliance-events/v1"]
    }
  }
}
```

---

## Metadata Attachment

AFIX metadata attaches at two granularities: per-message and per-task.

### Per-Message: Compliance Events

Compliance Events attach to `message.metadata` on every A2A `Message` object. Each event in the array records an atomic governance action at the time it occurred.

```json
{
  "role": "agent",
  "parts": [{ "text": "Portfolio analysis complete." }],
  "metadata": {
    "https://finos.org/afix/compliance-events/v1": [
      {
        "eventId": "evt_7f3a2b1c",
        "eventType": "DATA_ACCESS",
        "timestamp": "2025-03-15T14:23:01Z",
        "agentId": "agt_portfolio_analyzer_v2",
        "regulatoryBasis": "MiFID II Article 25",
        "dataClassification": "MNPI",
        "approved": true,
        "approver": "agt_compliance_gate_v1"
      }
    ]
  }
}
```

### Per-Task: Liability and Entitlement

Liability Metadata and Entitlement Interchange attach to `task.metadata`. They are set at task initiation and may be updated at task completion.

```json
{
  "id": "task_8d4e9f2a",
  "status": { "state": "completed" },
  "metadata": {
    "https://finos.org/afix/entitlement/v1": {
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
    "https://finos.org/afix/liability/v1": {
      "decisionId": "dec_9c3f1a7e",
      "agentChain": ["agt_orchestrator_v1", "agt_portfolio_analyzer_v2"],
      "humanInLoop": false,
      "recommendationType": "INVESTMENT_ADVICE",
      "liabilityOwner": "agt_orchestrator_v1",
      "firmLiabilityAccepted": true,
      "regulatoryDisclosureRequired": true
    }
  }
}
```

---

## Task State Invariants

Four machine-verifiable governance invariants must hold at task completion. Servers must validate these before transitioning a task to `completed` state; failure transitions the task to `failed` with a structured error.

| Invariant | Rule | Failure Action |
|---|---|---|
| **Event Completeness** | Every `DATA_ACCESS` compliance event must have a corresponding `DATA_RELEASE` event if data was returned | Reject task completion |
| **Liability Closure** | `liability.liabilityOwner` must be set before task reaches `completed` state | Reject task completion |
| **High-Risk Escalation** | If `agentCard.riskTier == "HIGH"` and `liability.humanInLoop == false`, a human escalation compliance event must be present | Reject task completion |
| **Entitlement Pre-flight** | All `dataAssets` referenced in entitlement must have non-expired `expiresAt` values at task initiation | Reject task initiation |

---

## AP2 Coexistence

AFIX extensions coexist with AP2 (A2A Policy Protocol) extensions in the same `capabilities.extensions[]` array without conflict. Each extension uses a distinct URI namespace; no field names overlap. A compliant agent may declare both AP2 and AFIX extensions simultaneously.

```json
{
  "capabilities": {
    "extensions": [
      {
        "uri": "https://a2a-policy.org/ap2/v1",
        "required": true,
        "params": { "policySet": "financial-services-baseline" }
      },
      {
        "uri": "https://finos.org/afix/agent-card/v1",
        "required": true,
        "params": { "riskTier": "HIGH", "regulatoryJurisdictions": ["SEC"] }
      }
    ]
  }
}
```

---

## Sources

1. Google Agent-to-Agent Protocol Specification — [https://google.github.io/A2A/](https://google.github.io/A2A/)
2. A2A Agent Card Schema — [https://google.github.io/A2A/specification/#agent-card](https://google.github.io/A2A/specification/#agent-card)
3. A2A Extension Mechanism — [https://google.github.io/A2A/specification/#extensions](https://google.github.io/A2A/specification/#extensions)
4. A2A Task and Message Object Definitions — [https://google.github.io/A2A/specification/#task-object](https://google.github.io/A2A/specification/#task-object)
5. FINOS Common Cloud Controls for AI (CC4AI) — [https://www.finos.org/common-cloud-controls](https://www.finos.org/common-cloud-controls)
