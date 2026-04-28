# AFIX: Agentic Financial Interoperability eXchange

A protocol-agnostic governance interchange format for cross-enterprise agent interactions in financial services.

---

## What is AFIX?

AFIX defines a shared vocabulary for agent governance at firm boundaries. It specifies what structured metadata must cross when a governed agent in one institution calls a governed agent in another — without prescribing how either firm governs itself internally. AFIX is transport-neutral: it provides binding specifications for both Google's Agent-to-Agent (A2A) protocol and Anthropic's Model Context Protocol (MCP). The goal is interoperability, not uniformity.

---

## The Problem

Bloomberg, FactSet, and LSEG have each independently built equivalent governance middleware layers around MCP — covering identity, access control, audit, and rate limiting. JPMC, Morgan Stanley, and BlackRock all have active AI governance programs. Framework-level coordination exists through bodies like FINOS CC4AI. What does not exist is any coordination on governance interchange: when your governed agent calls my governed agent, there is no shared vocabulary for the metadata that should accompany that call. Every integration currently requires bespoke negotiation of what identity means, what an audit event looks like, and who carries liability. AFIX proposes to fix that.

---

## Schema Components

| Component | Description | Schema |
|---|---|---|
| **Agent Card** | Organizational identity, governance posture, regulatory jurisdiction, supply chain attestation | [`afix-agent-card-v1.0.0.json`](/schemas/afix-agent-card-v1.0.0.json) |
| **Compliance Event** | Universal audit primitive — six-field interop core with regulatory compliance extension tier | [`afix-compliance-event-v1.0.0.json`](/schemas/afix-compliance-event-v1.0.0.json) |
| **Entitlement Interchange** | Cross-enterprise data/service access rights with conjunctive/disjunctive grouping | [`afix-entitlement-v1.0.0.json`](/schemas/afix-entitlement-v1.0.0.json) |
| **Liability Metadata** | Multi-party liability allocation with ISDA-analog bilateral and EMV proportional shift | [`afix-liability-metadata-v1.0.0.json`](/schemas/afix-liability-metadata-v1.0.0.json) |
| **Governance Hop Record** | Per-hop audit trail for multi-agent chains with LEI-based institutional identity | [`afix-governance-hop-record-v1.0.0.json`](/schemas/afix-governance-hop-record-v1.0.0.json) |

---

## Protocol Bindings

**A2A Binding** — AFIX extensions attach natively to the A2A Agent Card `capabilities.extensions[]` array and to per-message and per-task metadata fields. Uses protocol-native extension negotiation via the `X-A2A-Extensions` header. See [`/schemas/extensions/a2a-binding-v1.0.0.md`](/schemas/extensions/a2a-binding-v1.0.0.md).

**MCP Binding** — AFIX registers as a named capability extension under SEP-2133 (`finos.org/afix-governance`), with a `_meta` fallback convention for servers not yet on SEP-2133, and gateway enforcement patterns for Kong, Apigee, and custom proxy deployments. See [`/schemas/extensions/mcp-binding-v1.0.0.md`](/schemas/extensions/mcp-binding-v1.0.0.md).

---

## Research

The `/research` directory contains the full methodology behind AFIX: source syntheses, stress-tests, steelmanned counterarguments, and design decision rationale with citations. Every claim is sourced. Every assumption is tested. Published for transparency and to support community review.

---

## Quick Start

1. Read the [Agent Card schema](/schemas/afix-agent-card-v1.0.0.json) — this is the minimum viable governance artifact.
2. Pick your transport: [A2A binding](/schemas/extensions/a2a-binding-v1.0.0.md) or [MCP binding](/schemas/extensions/mcp-binding-v1.0.0.md).
3. Start with Agent Cards only. Full compliance event streams and liability chains can be added incrementally once identity and capability negotiation is working.
4. Add Compliance Events, Entitlement Interchange, and Liability Metadata as your integration matures.

Governance Hop Records are only required for multi-hop chains where intermediate agents must attest to governance state.

---

## Relationship to Blog Series

AFIX was developed as part of a research series on agentic AI governance in financial services. The series covers the emergence of multi-agent systems, the governance gap at firm boundaries, and the case for standardized interchange formats. [Blog Series](https://ashaykubal.com).

---

## Status

Research proposal. Not a standard. Not production-tested. Published for community review, feedback, and potential standardization under FINOS or a similar open governance body. Schema versions are tagged; breaking changes will increment the major version.

---

## License

Apache 2.0. See [LICENSE](/LICENSE).

---

## Contributing

Open an issue or start a discussion. The schemas are designed to be forked and adapted — institutional requirements vary, and AFIX is intended as a baseline, not a ceiling.
