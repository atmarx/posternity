# Utility tier

Separate small VMs — never one consolidated utility host. Blast radius, independent patching, and each is a named SSP component with its own hardening line:

| VM | Role |
|---|---|
| **Keycloak** | Enclave identity provider; brokers the institutional IdP; local break-glass realm |
| **Guacamole** (+guacd) | Tenant access gateway; session recording |
| **Forgejo** | Library metadata, runbooks, tenant automation repos, CI |
| **Zabbix** | Availability monitoring, tenant self-service dashboards |
| **Syslog aggregator** | rsyslog/syslog-ng; collects all enclave logs; single TLS egress to enterprise log service |
| **Media share** | nginx (HTTP) and/or NFS; serves ISOs and firmware to BMCs and jump VMs |

On a two-node cluster, affine utility VMs opposite the active pfSense instance where sensible; Proxmox HA restart is the failover story.
