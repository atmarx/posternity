# System Security Plan skeleton

Mapped to **NIST SP 800-171** (an 800-53 moderate crosswalk may be appended for audiences that prefer it). Section numbering below is the SSP's own.

## 10.1 System identification and description

Name, owner, operational status, system type: **management enclave / security support system**. Key framing sentence: *this system processes no research data; it carries administrative access to systems that may.*

## 10.2 System boundary and environment

The boundary diagram is the heart of the document. **In scope:** pfSense pair, Proxmox hosts (hypervisor substrate), the six utility VMs, tenant jump VMs, switch management-VLAN configuration, the BMCs as connected components, the Headscale termination. **Out of scope:** research servers' production interfaces and data planes. **Interconnections:** campus network (user ingress), institutional IdP, institutional smarthost, enterprise log service, campus NTP.

## 10.3 Security categorization

Confidentiality of enclave *contents* is low (no research data). **Integrity and availability of the management plane are high**: compromise of the enclave is compromise-by-proxy of every connected research system; loss of the enclave removes out-of-band recovery capability precisely when it is needed. Categorize on the impact of the systems the enclave controls, not the data it carries.

## 10.4 Roles and responsibilities

System owner; platform operators; tenant "designated technical contacts" and the scope of what they are entrusted with; PIs as access requesters/attesters; the enterprise log service as independent audit witness; the escrow custodian.

## 10.5 Data flows (each diagrammed)

(a) tenant → Guacamole → jump VM / SSH → BMC; (b) operator → Headscale (pfSense) → any enclave segment including the hypervisor-BMC tier; (c) BMC → internal services (NTP/DNS via pfSense; monitoring poll inbound from Zabbix) → nothing else; (d) audit: components → syslog aggregator → enterprise log service. Plus the push-in library refresh flow.

## 10.6 Control implementation by family

| Family | Strongest evidence |
|---|---|
| **AC** (3.1.x) | Per-tenant VLAN + deny-by-default pfSense rules as *technical* enforcement of least privilege; the testable invariant "no tenant can reach another tenant's segment"; groups-claim authorization at every service; explicit named grants narrower than research-group membership; IdP-backstop deprovisioning layered under local group hygiene |
| **IA** (3.5.x) | OIDC → enclave Keycloak → institutional IdP with institutional MFA on the tenant path; SSO-backed, key-expiring operator identities; ePPN identifiers; no shared accounts except escrowed break-glass |
| **AU** (3.3.x) | Full session recording of every management action; unified TLS syslog to an independent enterprise witness; local working-copy retention; session metadata correlation; recordings as nonrepudiable evidence |
| **MA** (3.7.x) | **The system *is* the MA family's implementation.** 3.7.5 (multifactor for nonlocal maintenance) is satisfied for every connected research system by the enclave itself. Frame the enclave as a **control provider from which connected systems' SSPs inherit** — research groups' own compliance documents point here rather than re-deriving maintenance controls |
| **SC** (3.13.x) | Boundary protection with deny-all egress and four enumerated exceptions; IPMI protocol containment (RMCP+ never crosses the boundary; cipher 0 disabled; Redfish preferred); TLS on the audit stream; WireGuard on the operator path; CARP HA as availability protection |
| **CM** (3.4.x) | BMC intake hardening baseline (scripted, logged); pfSense configuration and golden-image builds under version control in Forgejo; controlled intake as the sole path onto the network; monthly jump-VM rebuilds from a version-controlled baseline; library allowlist curation with commit-recorded changes |
| **SI** (3.14.x) | Manifest↔share hash reconciliation; fleet firmware-currency triggers from Zabbix vs. library catalog; new-MAC detection; push-in refresh with outside-the-boundary human verification |
| **IR / CP** (3.6.x) | BMC credential-compromise playbook; layered break-glass down to physical console; full-cluster-loss recovery narrative; tenant-dissolution decommission checklist |

## 10.7 Risk acceptances

[Bidirectional clipboard/SFTP](../security/risk-contingency.md#risk-acceptance-bidirectional-clipboard-and-sftp), stated with rationale, compensating control, and residual-risk assessment. Any others accumulated during deployment.

## 10.8 Continuous monitoring

The [attestation schedule](../operations/attestations.md), the detective controls ([roster reconciliation](../access/identity.md#detective-reconciliation-optional-module), [manifest reconciliation](../library/curation.md#manifest-share-reconciliation), [Zabbix detective functions](../security/audit.md#zabbix)), and firmware currency as an ongoing metric.

## 10.9 Contingency and POA&M

The layered [break-glass narrative](../security/risk-contingency.md#break-glass-layers-outermost-first), the [recovery walk-through](../security/risk-contingency.md#availability-notes), and known gaps with remediation dates.
