# Decisions ledger

The load-bearing decisions and why. (Full discussion history lives with the project.)

| # | Decision | Rationale (compressed) |
|---|---|---|
| 1 | Per-tenant VLAN, /26 groups, /24 clusters, MAC-reserved DHCP | Least privilege as network fact; reservation table doubles as inventory; no renumbering |
| 2 | Per-group jump VM behind Guacamole as the **sole tenant path** | A real browser inside the segment handles every BMC UI/KVM/virtual-media quirk; automation moves inside and gets recorded; eliminates the tenant-VPN use case |
| 3 | Guacamole session recording, mandatory | Nonrepudiable record of every management action; the AU family's centerpiece |
| 4 | No tenant VPN; **Headscale on pfSense for operators only** | Two identity-backed doors beat "everyone has a VPN option"; SSO + key expiry fix raw WireGuard's lifecycle problem; short attestable peer list |
| 5 | Dedicated enclave Keycloak; institutional IdP upstream; **no research-platform identity chain** | Boundary independence; the enclave must work during a research-platform incident; separate failure/incident domains |
| 6 | Local Keycloak groups authoritative; explicit named enrollment; research rosters demoted to **detective cross-check** | Management access is narrower than data access; roster-mirroring is privilege inflation; drift detection without dependency |
| 7 | One groups claim drives Guacamole + Forgejo + Zabbix | Single authorization source; zero per-service rosters |
| 8 | Forgejo over Gitea over GitLab CE | OIDC group→org mapping free (Premium-gated in GitLab); lightweight; community governance; Actions suffice for CI |
| 9 | Library split: Forgejo metadata + plain HTTP/NFS media share + hash reconciler | BMCs need dumb URLs; git stays lean; manifest↔share drift becomes a detection event |
| 10 | Zero egress; four enumerated exceptions; **push-in** library refresh with outside verification | Kills exfil, C2, and supply-chain drift at once; trivially auditable; human verification happens outside the boundary |
| 11 | Bidirectional clipboard/SFTP as documented risk acceptance | No research data in the boundary; blocking punishes the high-stress legitimate user; screen-reading defeats the control anyway; session recording compensates |
| 12 | Audit = one syslog/TLS stream to the enterprise witness; recordings local; **local retention kept** | Downstream offered custody — take it; their filter may drop your incident's key line, so keep the working copy |
| 13 | pfSense serves NTP/DNS; **BMC egress = zero** | All BMC dependencies satisfied internally; deny-all stays clean |
| 14 | Zabbix central; tenant RW on own host group minus host lifecycle; **Vault-backed read-only BMC creds** | Self-service monitoring without CM collision; the biggest secret-holder holds only read-only creds fetched at use-time |
| 15 | Separate small utility VMs; QDevice; CARP+XMLRPC+DHCP-failover | Blast radius; quorum on two nodes; stateful failover |
| 16 | Break-glass escrowed with independent custodian; hypervisor BMCs on operator-only tier; physical-console floor | Circularity named and answered in layers; independent escrow > self-escrow |
| 17 | Monthly golden-image jump-VM rebuilds; durable tenant material in Forgejo orgs | Anti-snowflake; CM evidence generates itself |
| 18 | Naming: Posternity (project) / per-institution deployment names / boring internal slugs | Reference-architecture layering mirrors the codebase split; collision-resistant; SSP cover pages stay professional |
