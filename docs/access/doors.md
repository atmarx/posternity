# The two doors

Exactly two doors. Both identity-backed. Nothing else.

## Tenant path — Guacamole

Browser → **Apache Guacamole** → (OIDC) → enclave Keycloak → institutional IdP (with institutional MFA) → the tenant's jump VM (RDP/VNC) and/or SSH targets within the tenant's VLAN only.

- **Every session is recorded** server-side: screen + input for graphical sessions, full typescript for SSH. Recordings are stored on a dedicated volume the user cannot reach, with [defined retention](../security/audit.md#session-recordings). Session *metadata* (who, which connection, when, duration) flows into the syslog stream.
- Connection visibility is driven by the Keycloak [**groups claim**](identity.md): a tenant sees only their own connections. No Guacamole local accounts exist except the documented [break-glass account](../security/risk-contingency.md#break-glass-layers-outermost-first).
- **Clipboard and SFTP are bidirectional** — a documented [risk acceptance](../security/risk-contingency.md#risk-acceptance-bidirectional-clipboard-and-sftp), not an oversight.
- **The half-screen doctrine** (user guide language): *the management window is for managing; your local browser is for searching.* Tenants troubleshoot with Guacamole on one half of the screen and their own local browser on the other. General web access does not exist inside the enclave and is not missed.

## Operator path — Headscale

**Headscale** (self-hosted Tailscale control plane) with the WireGuard data path **terminated on pfSense**. Available to platform operators only.

- Identities are **SSO-backed** (Headscale OIDC against the enclave Keycloak → institutional IdP) with **key expiry** — solving the two structural weaknesses of raw WireGuard (no MFA, no lifecycle) with infrastructure already in the stack.
- The operator peer list is short by design and [attested periodically](../operations/attestations.md).
- Operators use this path for: library refresh ([push-in](../library/curation.md)), hypervisor and switch management, the hypervisors' own BMCs ([break-glass tier](../security/risk-contingency.md#break-glass-layers-outermost-first)), and incident response when the utility tier is degraded.

**Design consequence:** there is no tenant VPN. The per-group jump VM eliminated the automation use case that traditionally forces one; Guacamole SFTP and the internal library cover file movement; the operator mesh covers privileged work. "Two doors — one for tenants, one for operators, both identity-backed" is the boundary story.
