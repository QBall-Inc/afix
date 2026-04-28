# Changelog

All notable changes to AFIX schemas are documented here.

## [1.0.0] — 2026-04-26

### Initial Release

- **Agent Card** (v1.0.0): Organizational identity, LEI, regulatory jurisdiction, compliance certifications (ISO 42001 tier mapping), authorized actions, escalation chain, supply chain attestation (AI-BOM reference, model provenance), interop declarations.
- **Compliance Event** (v1.0.0): Six-field interoperability core (event_id, event_type, timestamp, actor, target, action) plus authorization and outcome. Domain extensions via `authorization._ext_refs`. Separate regulatory compliance extension tier (9 fields for EU AI Act, MiFID II, SEC 17a-4).
- **Entitlement Interchange** (v1.0.0): Conjunctive/disjunctive entitlement grouping (RIXML-inspired), data classification, redistribution rights (Bloomberg/FactSet/LSEG licensing model), rate limiting, emergency revocation.
- **Liability Metadata** (v1.0.0): Five liability models, ISDA-analog bilateral primary frame, EMV proportional shift mechanism via governance certification tiers, dispute resolution with evidence chain.
- **Governance Hop Record** (v1.0.0): Append-only per-hop governance chain with LEI-based institutional identity, hash-linked integrity, compensation definitions, SWIFT UETR-analog correlation.
- **Governance Profile** (v1.0.0): Capability/Extension/Profile pattern (UCP-inspired). Selective capability support, six domain extensions, transport bindings.
- **A2A Binding** (v1.0.0): Five protocol-native extensions, `X-A2A-Extensions` header negotiation, per-message and per-task metadata, four task state invariants.
- **MCP Binding** (v1.0.0): SEP-2133 extension registration (primary), `_meta` conventions (fallback), gateway enforcement patterns (Kong, Apigee, custom proxy), dual discovery mechanisms.

### Design Decisions

- `ap2_mandate_ref` placed in `ext/agentic-commerce` domain extension, not base schema (protocol-agnosticism principle).
- Compliance Event split into interop core + regulatory extension (six fields stable, nine fields absorb regulatory change).
- ISDA bilateral agreements as primary liability frame; EMV as secondary mechanism reference.
- LEI as identity anchor (centrally issued by GLEIF, no new registry).
- ISO 42001 certification tiers in governance certification levels.
