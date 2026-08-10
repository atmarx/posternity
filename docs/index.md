# Posternity

**An out-of-band management enclave reference architecture for research computing**

*Specification draft 0.1*

Research computing fleets accumulate baseboard management controllers (BMCs) — iDRAC, iLO, OpenBMC, Supermicro BMC — one per server, each a small always-on computer with total power over its host: power control, console access, virtual media, firmware, sensors. BMCs are chronically under-secured (weak default credentials, slow firmware patching, a protocol with known cryptographic defects) and chronically over-exposed (flat management VLANs reachable by whole departments, or worse).

Posternity defines a **dedicated out-of-band (OOB) management enclave** that:

1. **Contains** all BMC traffic within a bounded, segmented network that no research data plane touches.
2. **Segments** tenants (research groups, clusters) so each can reach only its own BMCs.
3. **Gates** all human access through two identity-backed doors: a recorded web gateway for tenants, and an SSO-backed mesh VPN for platform operators.
4. **Audits** everything: full session recording, unified syslog egress, detective reconciliation.
5. **Curates** a software and media library so tenants never need to bring their own tools — and the enclave never needs general internet egress.
6. **Documents** itself as a System Security Plan mapped to NIST SP 800-171, positioned as a control *provider* from which connected systems' own security plans inherit.

## Scope

**In scope:** the enclave network, its firewall/routing layer, its utility services, tenant jump hosts, the BMCs as connected components, and all access paths.

**Explicitly out of scope:** the research servers' production (in-band) interfaces, operating systems, and data planes. The enclave carries *administrative access to* systems that may process research data; it processes **no research data itself**. This claim bounds the [security categorization](ssp/index.md#103-security-categorization) and justifies several [deliberate control decisions](security/risk-contingency.md).

## Naming and repository topology

**Posternity** is the project: this specification, the sample codebase, and the reference architecture. Institutions deploy Posternity under their own deployment names.

| Layer | Name | Repository | Visibility |
|---|---|---|---|
| Project / reference architecture | Posternity | `atmarx/posternity` | Public |
| Reference deployment (fictional Northwinds University) | **{{ deployment.name }}** (`{{ deployment.base_domain }}`) | `{{ deployment.repo }}` | Public |
| Institutional deployment (example) | *institution's choice* (e.g., `mgmt-enclave`) | private | Private |

The Northwinds **Mudroom** deployment is the worked example threaded through this documentation: it demonstrates what a completed deployment looks like, with every institution-specific value filled in with fictional-but-realistic choices. Real institutions clone `posternity`, copy the structure of `mudroom`, and supply their own values via the [deployment overlay](reference/deployment-parameters.md).

**Contribution discipline:** anything an institutional deployment needs that is not institution-specific is built in `posternity` and consumed downstream — never patched locally. The moment a private deployment carries a generalizable fix the reference does not, the reference architecture has started lying.

!!! note "Why \"Posternity\"?"
    A *postern* is the small, guarded rear gate of a fortification — the private entrance used by those who belong there. The reference architecture is also the durable artifact written *for posterity*, while deployments are its ephemeral instantiations. Northwinds calls their deployment the **Mudroom**: the utility entrance to the house that holds the Hearth — where the muddy work happens, tools and deliveries come in, and nothing dirty tracks into the living space.

## Reading order

The navigation follows the specification: the [architecture](architecture/index.md), the [two doors and identity model](access/doors.md), the [software and media library](library/index.md), the [security posture](security/egress.md), [lifecycle operations](operations/onboarding.md), and the [SSP skeleton](ssp/index.md). The [decisions ledger](reference/decisions.md) records why each load-bearing choice was made.
