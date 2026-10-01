# Deployment parameters

Every institution-specific value a deployer must supply lives in one file: [`overlay/deployment.yml`](https://github.com/atmarx/posternity/blob/main/overlay/deployment.yml). That file is the **overlay** — an institutional deployment repository overrides it with real values and rebuilds this site, so the walkthrough you are reading becomes that institution's own deployment documentation.

This build renders the fictional Northwinds University **{{ deployment.name }}** deployment as its worked example; the values below come straight from the overlay file.

| Parameter | Description | This deployment ({{ deployment.name }}) |
|---|---|---|
| Deployment name | Human name for the deployment | {{ deployment.name }} |
| Base domain | DNS zone for enclave services | `{{ deployment.base_domain }}` |
| Institutional IdP | OIDC/SAML endpoint brokered by enclave Keycloak | `{{ deployment.idp }}` |
| MFA source | Where multifactor is enforced | {{ deployment.mfa_source }} |
| SMTP smarthost | Destination for Zabbix alerts; auth basis | {{ deployment.smtp_smarthost }} |
| Enterprise log service | Syslog/TLS destination; custodian | {{ deployment.log_service }} |
| Campus NTP | pfSense upstream time | `{{ deployment.campus_ntp }}` |
| Escrow custodian | Holder of break-glass credentials | {{ deployment.escrow_custodian }} |
| Subnet plan | Enclave supernet; per-tenant /26 map; cluster /24s; utility and operator VLANs | `{{ deployment.supernet }}` ({{ deployment.subnet_plan }}) |
| VLAN plan | VLAN IDs per tenant/utility/operator segment | {{ deployment.vlan_plan }} |
| Certificate authority | TLS issuance for enclave services | {{ deployment.certificate_authority }} |
| Hypervisor platform | Substrate + HA specifics | {{ deployment.hypervisor_platform }} |
| Fleet vendors | Determines firmware shelves + tooling | {{ deployment.fleet_vendors }} |
| Distro set | Installation media shelf | {{ deployment.distro_set }} |
| Research-group source of truth | Upstream for [detective reconciliation](../access/identity.md#detective-reconciliation-optional-module), if any | {{ deployment.group_source_of_truth }} |
| Recording retention | Session-recording schedule | {{ deployment.recording_retention }} |
| Log retention (local) | Aggregator working-copy window | {{ deployment.log_retention_local }} |
