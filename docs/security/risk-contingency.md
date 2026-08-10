# Risk acceptances and contingency

## Risk acceptance: bidirectional clipboard and SFTP

Guacamole clipboard and SFTP are bidirectional on all connections. **Rationale:** (1) the enclave contains **no research data** — tenant jump VMs reach BMCs, not data planes; the clipboard moves error messages and configuration text between a user and their own screen. (2) The realistic blocking scenario punishes exactly the wrong person: tenant sessions occur predominantly when in-band access is already broken — high-stress moments when a user urgently needs to move an error message into a search engine or an LLM. (3) Copy-out restrictions are defeated by trivially available screen-reading (an LLM reading the Guacamole half of the screen reproduces the text with zero effort), converting the control into pure friction with no security value. **Compensating control:** full session recording captures everything transiting the human channel. **Residual risk:** assessed low; accepted with this documented rationale. (Assessors respect a reasoned acceptance more than a checkbox control that a screenshot defeats.)

## The circularity problem, named

The enclave's control plane (Keycloak, Guacamole, Forgejo, Zabbix) runs on the Proxmox cluster — infrastructure the enclave exists to help manage. If the cluster is unhealthy, the tools used to reach its BMCs are down, and so is the IdP. Every management plane has a bootstrapping story; this one is layered explicitly.

## Break-glass layers (outermost first)

1. **Operator mesh survives VM death:** Headscale/WireGuard terminates on pfSense, not on a VM.
2. **Hypervisor BMCs on an operator-only segment:** the Proxmox nodes' own BMCs (and switch management interfaces) live in a dedicated VLAN reachable only via the operator path — the "management of the management" tier.
3. **Local break-glass accounts** on Guacamole and pfSense — one documented local admin each, existing precisely because Keycloak may be unreachable. Credentials **escrowed with an organizationally independent custodian** (e.g., the enterprise IT credential vault) — independent escrow is stronger than self-escrow and doubles as the DR story.
4. **Physical console procedure** as the floor: documented, tested, in the runbook.

## Availability notes

CARP handles firewall failover; XML-RPC sync + DHCP failover preserve state; the QDevice preserves Proxmox quorum on single-node loss; utility VMs restart via Proxmox HA on the surviving node. The [contingency section of the SSP](../ssp/index.md#109-contingency-and-poam) narrates a full-cluster-loss recovery: physical console → single pfSense instance → operator mesh → restore utility tier from Proxmox backups (Forgejo holds all configuration as code).
