# ACGP Use Case 1: Research Workflow Pilot
_Date: 2026-04-29 | Status: Design — Not Yet Executed_

---

## Purpose

This document specifies the first ACGP (Agentic Cross-Enterprise Governance Pilot) use case: a cross-firm research workflow that validates whether AFIX governance metadata propagates correctly across enterprise boundaries using A2A and MCP as they exist today. The pilot has not been executed. It is a design document specifying how participating firms would validate AFIX governance propagation in a realistic financial services scenario.

The success criterion is narrow and falsifiable: governance metadata — identity, compliance events, entitlements, liability attribution, and hop provenance — propagates intact across three firm boundaries through both protocol transports. If that works, everything else is an application layer on top of a working interchange.

---

## 1. Pilot Architecture

### 1.1 Scenario

A sell-side institution delegates a research task that requires data from an external vendor, with the enriched output consumed by a buy-side client. The workflow crosses three enterprise boundaries, exercises both A2A and MCP transports, and requires auditable governance provenance at each transition. This is a routine workflow in capital markets — what makes it a governance pilot is the requirement that every firm boundary crossing carries machine-readable governance metadata using the AFIX schema vocabulary.

### 1.2 Participating Entities

| Entity | Role | Transport(s) | AFIX Components Active |
|--------|------|-------------|------------------------|
| **Firm A** — Sell-side institution (e.g., JPMC-scale) | Orchestrator agent delegating research tasks across counterparties | A2A (peer negotiation with Firm C), MCP (tool calls to Firm B) | Full stack: Agent Card, Compliance Events, Entitlement, Liability, GHR |
| **Firm B** — Data vendor (e.g., Bloomberg-scale) | Exposes market data services via MCP with governance middleware | MCP (tool exposure via gateway) | Agent Card, Compliance Events, Entitlement |
| **Firm C** — Buy-side institution (e.g., BlackRock-scale) | Consumes enriched research output; needs auditable governance trail for EU AI Act regulatory reporting | A2A (peer negotiation with Firm A) | Full stack: Agent Card, Compliance Events, Entitlement, Liability, GHR |
| **Governance Observer** (independent auditor) | Post-hoc audit reconstruction; not grading own homework | Read-only on both transports | Compliance Events (read), Liability (read), GHR (read) |

### 1.3 Fictional Entity References

For consistency with the article and schema examples, the pilot uses the following fictional entities:

| Entity | Legal Name | LEI | Jurisdiction |
|--------|-----------|-----|-------------|
| Firm A | Meridian Capital LLC | 5493001KJTIIGC8Y1R12 | SEC, FINRA |
| Firm B | Apex Market Data Corp | 2138002LKWSU84OC9E37 | SEC, FCA |
| Firm C | Vanguard Ridge Partners | 9845003MPQTZ67FN4H51 | SEC, FINRA, MiFID II |

---

## 2. Five-Phase Flow

### Phase 1 — Discovery

Before any research interaction begins, agents discover each other and exchange governance postures.

**Firm A → Firm B (MCP path):** Firm A's orchestrator agent would query Firm B's AFIX Agent Card via the `afix_agent_card` MCP tool exposed through Firm B's governance middleware. The response carries Firm B's LEI, ISO 42001 certification tier, authorized data categories, and entitlement requirements.

**Firm A → Firm C (A2A path):** Firm A fetches Firm C's Agent Card from `/.well-known/agent.json`, reading the `https://finos.org/afix/v1/agent-card` extension parameters.

Both parties evaluate: regulatory jurisdiction compatibility, compliance certification status (ISO 42001 current? SOC2 Type II?), authority thresholds, and supported protocol bindings.

Firm A's policy engine evaluates whether each counterparty's governance posture meets its standards — the same class of pre-trade credit check that exists in every electronic trading relationship, applied to agent interactions instead of order flow.

**Example — Firm B Agent Card response (MCP):**

```json
{
  "schema_version": "1.0.0",
  "organization": {
    "legal_name": "Apex Market Data Corp",
    "lei": "2138002LKWSU84OC9E37"
  },
  "regulatory_jurisdiction": ["SEC", "FCA"],
  "compliance_certifications": [
    {
      "framework": "ISO_42001",
      "status": "CURRENT",
      "certification_body": "BSI Group",
      "valid_until": "2027-03-15"
    },
    {
      "framework": "SOC2_TYPE_II",
      "status": "CURRENT",
      "valid_until": "2026-12-01"
    }
  ],
  "governance_certification_level": "TIER_3_FULL",
  "authorized_actions": [
    {
      "action_type": "DATA_ACCESS",
      "asset_classes": ["equity", "fixed_income", "derivatives"],
      "data_classification": "MARKET_DATA",
      "redistribution": "NON_DISPLAY_OK"
    }
  ],
  "escalation_chain": [
    { "level": 1, "role": "data-ops-lead", "channel": "automated" },
    { "level": 2, "role": "chief-data-officer", "channel": "manual" }
  ]
}
```

**Compliance Events emitted:** `TASK_INITIATED` (Firm A), `ENTITLEMENT_VERIFIED` (all parties verify mutual entitlements).

---

### Phase 2 — Negotiation

Two transport paths, exercised in parallel within the same pilot run.

**Over A2A (Firm A ↔ Firm C):** AFIX extensions negotiated via Agent Card header handshake. Firm A's Agent Card declares five extensions in the `extensions[]` array — four required (Agent Card, Compliance Event, Entitlement, Liability) and one optional (GHR). Firm C validates that all `required: true` extensions are present. If not, Firm C rejects with JSON-RPC error `-32001` before task creation.

**Example — A2A extension negotiation (Firm A → Firm C):**

```json
{
  "extensions": [
    {
      "uri": "https://finos.org/afix/v1/agent-card",
      "required": true,
      "params": {
        "organization": {
          "legal_name": "Meridian Capital LLC",
          "lei": "5493001KJTIIGC8Y1R12"
        },
        "governance_certification_level": "TIER_3_FULL"
      }
    },
    { "uri": "https://finos.org/afix/v1/compliance-event", "required": true },
    { "uri": "https://finos.org/afix/v1/entitlement", "required": true },
    { "uri": "https://finos.org/afix/v1/liability", "required": true },
    { "uri": "https://finos.org/afix/v1/ghr", "required": false }
  ]
}
```

**Over MCP (Firm A → Firm B):** SEP-2133 extension registration at connection initialization. If Firm B has not adopted SEP-2133, the `_meta` field fallback applies — governance metadata carried as convention rather than negotiated capability.

**Example — MCP SEP-2133 extension declaration (Firm A → Firm B):**

```json
{
  "method": "initialize",
  "params": {
    "extensions": [
      {
        "name": "finos.org/afix-governance",
        "version": "1.0.0",
        "components": [
          "agent-card",
          "compliance-event",
          "entitlement",
          "liability",
          "ghr"
        ],
        "required": true
      }
    ]
  }
}
```

The pilot exercises both paths because real-world adoption will not be uniform — some firms will adopt SEP-2133 in months, others in years.

---

### Phase 3 — Execution with Governance Propagation

This is the core of the pilot design: a three-hop governance chain across three enterprise boundaries.

**Hop 0 — Firm A originates the chain.** Firm A's orchestrator agent creates a Governance Hop Record (GHR) at chain origin, generating a `chain_id` — the correlation key that functions as a SWIFT Universal Electronic Transaction Reference (UETR) analog for agent governance.

**Hop 1 — Firm B executes data retrieval (MCP).** Firm A calls Firm B's `bloomberg_reference_data` tool (or equivalent) via MCP. The `_meta` field on the `CallToolRequest` carries the AFIX governance envelope:

```json
{
  "method": "tools/call",
  "params": {
    "name": "market_data_query",
    "arguments": {
      "securities": ["AAPL US Equity", "MSFT US Equity"],
      "fields": ["PX_LAST", "VOLUME", "PE_RATIO"]
    },
    "_meta": {
      "finos.org/afix/agent-card": {
        "lei": "5493001KJTIIGC8Y1R12",
        "agent_id": "research-orchestrator-3",
        "governance_certification_level": "TIER_3_FULL"
      },
      "finos.org/afix/compliance-event": {
        "event_id": "ce-2026-04-29-001",
        "actor": "research-orchestrator-3",
        "action": "DATA_ACCESSED",
        "target": "market_data_query",
        "timestamp": "2026-04-29T14:30:00Z",
        "authorization": {
          "mechanism": "ENTITLEMENT_GRANT",
          "grant_ref": "ent-meridian-apex-2026-001"
        },
        "outcome": "PENDING"
      },
      "finos.org/afix/entitlement": {
        "grant_ref": "ent-meridian-apex-2026-001",
        "grantor_lei": "2138002LKWSU84OC9E37",
        "grantee_lei": "5493001KJTIIGC8Y1R12",
        "scope": {
          "action_type": "DATA_ACCESS",
          "asset_classes": ["equity"],
          "data_classification": "MARKET_DATA",
          "redistribution": "NON_DISPLAY_OK"
        }
      },
      "finos.org/afix/ghr": {
        "chain_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
        "hop_index": 1,
        "actor_lei": "5493001KJTIIGC8Y1R12",
        "parent_hash": "a1b2c3...sha256-of-hop-0",
        "total_hops": 3
      }
    }
  }
}
```

Firm B executes the data retrieval, emits a Compliance Event recording the action (outcome: `SUCCESS`), and appends its own GHR entry with its LEI as `actor_lei`. The response `_meta` carries Firm B's Compliance Event and updated GHR.

**Hop 2 — Firm A forwards enriched output to Firm C (A2A).** Firm A takes Firm B's data response, enriches it with its own analysis, and forwards the result to Firm C over A2A. The A2A message metadata carries the full AFIX governance envelope:

```json
{
  "message": {
    "role": "agent",
    "parts": [{ "type": "text", "text": "Research output with enriched market data..." }],
    "metadata": {
      "https://finos.org/afix/v1/compliance-event": {
        "event_id": "ce-2026-04-29-003",
        "actor": "research-orchestrator-3",
        "action": "DATA_FORWARDED",
        "target": "vanguard-ridge-research-consumer",
        "timestamp": "2026-04-29T14:30:45Z",
        "authorization": {
          "mechanism": "ENTITLEMENT_GRANT",
          "grant_ref": "ent-meridian-vanguard-2026-001"
        },
        "outcome": "SUCCESS"
      },
      "https://finos.org/afix/v1/liability": {
        "liability_model": "PROPORTIONAL",
        "parties": [
          {
            "lei": "5493001KJTIIGC8Y1R12",
            "legal_name": "Meridian Capital LLC",
            "role": "ORIGINATOR",
            "governance_certification_level": "TIER_3_FULL"
          },
          {
            "lei": "2138002LKWSU84OC9E37",
            "legal_name": "Apex Market Data Corp",
            "role": "EXECUTOR",
            "governance_certification_level": "TIER_3_FULL"
          },
          {
            "lei": "9845003MPQTZ67FN4H51",
            "legal_name": "Vanguard Ridge Partners",
            "role": "CONSUMER",
            "governance_certification_level": "TIER_1_BASIC"
          }
        ],
        "default_assignment": { "rule": "PROPORTIONAL_BY_CERTIFICATION" }
      },
      "https://finos.org/afix/v1/ghr": {
        "chain_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
        "hop_index": 3,
        "actor_lei": "5493001KJTIIGC8Y1R12",
        "parent_hash": "d4e5f6...sha256-of-hop-2",
        "total_hops": 3,
        "compensation": {
          "type": "DATA_QUALITY_SLA",
          "timeout_days": 30,
          "description": "Meridian Capital guarantees data accuracy per SLA terms"
        }
      }
    }
  }
}
```

At each transition, `governance_certification_level` comparison would enable proportional liability attribution — a TIER_3_FULL certified firm assuming different liability weight than a TIER_1_BASIC counterparty.

---

### Phase 4 — Audit Reconstruction

Firm C's compliance team would pull the GHR chain by `chain_id` (`f47ac10b-58cc-4372-a567-0e02b2c3d479`). The reconstruction yields:

| Hop | Actor | LEI | Transport | Action | Certification |
|-----|-------|-----|-----------|--------|--------------|
| 0 | Meridian Capital LLC | 5493001KJTIIGC8Y1R12 | — (origin) | TASK_INITIATED | TIER_3_FULL |
| 1 | Apex Market Data Corp | 2138002LKWSU84OC9E37 | MCP | DATA_ACCESSED | TIER_3_FULL |
| 2 | Meridian Capital LLC | 5493001KJTIIGC8Y1R12 | MCP→A2A | DATA_FORWARDED | TIER_3_FULL |

Three hops, institutional identity verified at each, Compliance Events at each transition boundary. The governance trail would be self-contained — Firm C reconstructs complete provenance without trusting Firm A's internal logs.

**Governance Observer validation:** The independent auditor reconstructs the same governance state from three independent sources: (a) A2A message logs (Firm A ↔ Firm C), (b) MCP gateway logs (Firm A → Firm B), and (c) the GHR chain itself. All three reconstructions must yield identical governance conclusions.

---

### Phase 5 — Failure Case

Firm B's agent attempts an action outside its entitlement scope — redistributing derived analytics that the Entitlement Interchange restricts to `NON_DISPLAY_OK`.

**Expected behavior:**

1. Entitlement check fires denial at the governance middleware layer (Kong/Apigee gateway or MCP server-side validation).
2. A Compliance Event with `DENIED` outcome is emitted:

```json
{
  "event_id": "ce-2026-04-29-fail-001",
  "actor": "apex-data-agent-2",
  "action": "DATA_REDISTRIBUTED",
  "target": "unauthorized-third-party",
  "timestamp": "2026-04-29T14:31:15Z",
  "authorization": {
    "mechanism": "ENTITLEMENT_GRANT",
    "grant_ref": "ent-meridian-apex-2026-001",
    "denial_reason": "REDISTRIBUTION_SCOPE_EXCEEDED"
  },
  "outcome": "DENIED",
  "risk_classification": "LEVEL_2_HIGH_RISK"
}
```

3. The Compliance Event is pushed to the governance observer before the violation can propagate.
4. The governance layer prevents the unauthorized action, not just logs it.

**What this validates:** The AFIX entitlement model is enforceable at the protocol boundary, not merely advisory. The failure case is as important as the success case — a governance schema that logs violations but cannot prevent them provides audit value but not operational safety.

---

## 3. Success Criteria

### 3.1 Core Validation

| Criterion | Measurement | Target |
|-----------|-------------|--------|
| **Governance metadata propagation** | All five AFIX components present at each firm boundary crossing | 100% — no governance metadata dropped at any hop |
| **Schema identity across transports** | SHA-256 hash of each AFIX component schema is identical whether carried in A2A `message.metadata` or MCP `_meta` | 100% match — zero schema divergence |
| **Cross-transport chain integrity** | GHR chain that originates at Firm A, passes through Firm B (MCP), and terminates at Firm C (A2A) maintains unbroken hash chain | `parent_hash` on each hop correctly references SHA-256 of previous hop; `chain_id` matches throughout |
| **Audit reconstruction completeness** | Governance Observer reconstructs full governance state from protocol logs alone | Observer's reconstruction matches the governance state known to participating firms |
| **Entitlement enforcement** | Unauthorized actions blocked at protocol boundary | 100% of out-of-scope actions denied before execution |

### 3.2 Transport-Specific Validation

| Transport | Enforcement Test | Expected Behavior |
|-----------|-----------------|-------------------|
| **A2A** | Send `SendMessage` without required AFIX extensions to a `required: true` agent | Rejection with error code `-32001` before task creation |
| **MCP** | Send `tools/call` without `_meta` governance fields to a gateway-enforced server | 403 rejection at gateway before reaching MCP server |

### 3.3 Asymmetry Validation

The pilot's most intellectually significant test: does the same AFIX schema produce equivalent governance outcomes despite structurally different enforcement on each transport?

| Dimension | A2A Behavior | MCP Behavior | Equivalence Criterion |
|-----------|-------------|-------------|----------------------|
| Extension enforcement | Protocol-level rejection (`required: true`) | Gateway-level rejection (Kong/Apigee policy) | Both reject non-compliant clients; mechanism differs, outcome identical |
| Audit trail location | Self-documenting — governance metadata IN the message log | Infrastructure-documented — `_meta` + gateway logs | Governance Observer reaches same conclusions from either source |
| Multi-hop propagation | GHR appends to message metadata natively | Orchestrator must propagate `_meta` GHR between tool calls | Hash chain integrity identical; no governance data lost in MCP propagation |

---

## 4. What the Pilot Deliberately Scopes Out

| Excluded | Rationale | Where It Gets Resolved |
|----------|-----------|----------------------|
| Automated liability adjudication | Phase 1 is ISDA-analog bilateral only; automated adjudication requires consortium governance infrastructure that does not exist yet | Future ACGP phase or CEAG working group |
| Aggregator-scale event streaming | Requires answering "who operates the aggregator?" — a structural dependency beyond pilot scope | Phase 2 design, post-pilot |
| Cross-jurisdictional regulatory mapping | SEC/FINRA vs. FCA/MiFID II entitlement conflicts require regulatory interpretation, not schema validation | Separate workstream |
| Real-time execution-tier workflows | Matching engines, microsecond order routing — governed by existing FIX compliance fields, not AFIX | Explicitly out of scope per AFIX design boundary |

---

## 5. First-Adopter Incentive Analysis

The incentive structure mirrors FIX Protocol adoption in 1992. Salomon Brothers and Fidelity did not adopt FIX because an industry body told them to — they adopted it because they needed to reduce trade communication costs between two specific firms.

**Data vendor incentive (Firm B):** Standardized governance replaces per-client governance middleware integration with a single schema vocabulary. Today, Bloomberg, FactSet, and LSEG each independently build proprietary governance middleware for their MCP integrations. Each new client relationship requires bespoke governance configuration. AFIX reduces this to schema compliance — configure once, interoperate with any AFIX-compliant counterparty.

**Buy-side incentive (Firm C):** Auditable cross-firm governance trails that satisfy EU AI Act Article 12 record-keeping requirements without building bespoke integrations with every sell-side counterparty. One GHR chain query reconstructs complete provenance across all counterparties in a research workflow.

**Sell-side incentive (Firm A):** Demonstrable governance posture for agent-mediated services. As agent interactions cross firm boundaries, clients will increasingly require governance attestation as a precondition for counterparty relationships — the same pattern that drove FIX adoption from bilateral convenience to industry requirement.

---

## 6. Open Questions

| Question | Why It Matters | Where It Gets Resolved |
|----------|---------------|----------------------|
| Who maintains the chain correlation registry? | The GHR uses `chain_id` as a UETR analog, but SWIFT has a central tracker service. Without shared correlation infrastructure, each firm sees only its own hops. | Pilot Phase 2 — start with firm-local registries, evaluate shared registry need based on pilot results |
| What happens when the orchestrator drops `_meta` on MCP multi-hop? | If Firm A fails to propagate `_meta` governance fields from Firm B's response to the next tool call, the governance chain breaks silently. | Strongest weakness of MCP binding — may require "governance-aware orchestrator" certification |
| How does the Governance Observer access MCP gateway logs? | The pilot assumes the Observer can correlate MCP traces with gateway enforcement logs. Firms may not share gateway logs externally. | Requires defining a standard governance log export format, or mandating that enforcement decisions reflect back in response `_meta` |
| Does entitlement revocation propagate within SLA? | 30-second default SLA, but gateways cache policies. Actual propagation path from revocation event to gateway policy update is untested. | Requires testing with actual Kong/Apigee deployments |

---

## Sources

- [Tier 1] A2A Protocol Specification v1.0.0, https://a2a-protocol.org/latest/specification/
- [Tier 1] A2A Extensions, https://a2a-protocol.org/latest/topics/extensions/
- [Tier 1] MCP Specification (2025-06-18), https://modelcontextprotocol.io/specification/2025-06-18/server/tools
- [Tier 1] Stripe, "Optimizing payments at scale," 2025, https://go.stripe.global/rs/072-MDK-283/images/Optimizing-payments-at-scale.pdf
- [Tier 1] AFIX Schemas, https://github.com/QBall-Inc/afix/tree/main/schemas
